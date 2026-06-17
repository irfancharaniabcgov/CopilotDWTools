# Mode → References Mapping (Sub-Agent Pattern)

**Purpose**: Enable Sonnet orchestrator to route selective references to Haiku sub-agents, preventing context truncation (200K window).

**Strategy**: Each mode receives only the references it needs; orchestrator infers mode and routes with explicit checklist.

---

## Reference File Inventory

| File | Size (KB) | Purpose | Critical | Modes |
|---|---|---|---|---|
| kimball-patterns.md | 45 | Fact/dim grain, SCD, bus matrix | ✅ Yes | A, E, H, N |
| kimball-advanced-patterns.md | 30 | Data Vault, late-arriving facts, snapshots | ⚠️ Sometimes | A, H |
| sqlbi-dax-patterns.md | 110 | DAX measures, patterns, best practices | ✅ Yes | D, I, L |
| sqlbi-dax-patterns-advanced.md | 60 | Situational DAX (M2M, TREATAS, etc) | ⚠️ Sometimes | D, L |
| sqlbi-dax-patterns-niche.md | 25 | Rare patterns (currency, weighted avg) | ❌ No | [deferred] |
| ssas-tabular-bp.md | 65 | SSAS naming, relationships, partitions | ✅ Yes | B, I, M |
| dax-style-guide.md | 20 | DAX coding standards, VAR/RETURN | ⚠️ Sometimes | D, L |
| dax-studio-workflow.md | 18 | Server Timings, VertiPaq analysis | ❌ No (optional) | D |
| extended-properties-templates.md | 55 | sp_addextendedproperty scripts | ✅ Yes | C, H |
| documentation-authoring.md | 80 | Discovery-driven doc, coverage audit | ✅ Yes | [db-documenter] |
| dw-review-checklist.md | 35 | Findings report template, severity codes | ✅ Yes | A, B, C, D, O |
| dw-validation-patterns.md | 40 | T-SQL validation queries | ⚠️ Sometimes | A, O |
| dw-physical-design.md | 50 | Index strategy, partitioning | ✅ Yes | H, O |
| dw-calendar-build.md | 30 | Dimension.Calendar DDL + SP | ⚠️ Sometimes | H, I |
| elt-patterns.md | 65 | SSIS 4-package, SP conventions | ✅ Yes | F, J, K, M |
| source-system-analysis.md | 95 | Q1–Q10 queries, entity map, CSV discovery | ✅ Yes | P |
| ssdt-project-structure.md | 20 | SSDT layout, DACPAC patterns | ⚠️ Sometimes | H, M |
| ssas-deployment-processing.md | 25 | TE2, SSAS deployment, processing | ⚠️ Sometimes | I, M |
| tabular-editor-2-automation.md | 18 | TE2 CLI, BPA rules | ⚠️ Sometimes | I, M |
| devops-deployment-patterns.md | 55 | ADO Classic pipeline structure | ⚠️ Sometimes | G, M |
| devops-operations-patterns.md | 70 | Shared PS library, repo structure | ✅ Yes | G, M |
| security-implementation.md | 60 | RLS, OLS, least-privilege grants | ✅ Yes | B, I, M |
| pbirs-constraints.md | 35 | PBIRS limitations vs cloud PBI | ⚠️ Sometimes | B, G |
| pbix-report-standards.md | 25 | Debug tab, data freshness | ⚠️ Sometimes | [Power BI review] |
| cloud-migration-portability.md | 40 | Tabular → Fabric, SSIS → ADF | ⚠️ Sometimes | A, H, J |
| data-classification.md | 28 | Native SQL classification, taxonomy | ⚠️ Sometimes | C, H |
| dw-validation-patterns.md | 40 | Orphan facts, unknown members | ⚠️ Sometimes | A, O |
| performance-end-to-end.md | 50 | Query performance, cardinality | ⚠️ Sometimes | [perf review] |

**Total**: ~1,300 KB (~325K tokens if all loaded)

---

## Mode → References Mapping

### Mode A: DW Schema Review
**Input**: SQL Server connection OR DDL  
**Output**: Findings report (severity-coded)  
**Haiku tokens**: ~50K (5 references + checklist)

**Core references** (must load):
- kimball-patterns.md
- dw-review-checklist.md
- extended-properties-templates.md

**Conditional** (load if user has existing DW):
- kimball-advanced-patterns.md (if SCD Type 2 or snapshots exist)
- dw-validation-patterns.md (if user asks for validation queries)
- cloud-migration-portability.md (if user asks about Tabular → Fabric path)

**Do NOT load**:
- sqlbi-dax-patterns* (not used in schema review; only used in Mode D)
- elt-patterns (not used in schema review; only used in Mode F)
- pbix-report-standards, dax-studio-workflow (unrelated)

---

### Mode B: Tabular Model Review
**Input**: .bim file OR DMV output OR live connection  
**Output**: Findings report (severity-coded)  
**Haiku tokens**: ~45K (4 references + checklist)

**Core references** (must load):
- ssas-tabular-bp.md
- dw-review-checklist.md
- security-implementation.md

**Conditional** (load if applicable):
- sqlbi-dax-patterns.md (if measures present; review measure naming/structure only, not semantic)
- pbirs-constraints.md (if user asks about PBIRS-SSAS integration)

**Do NOT load**:
- kimball-patterns (schema review, not model review)
- elt-patterns, dw-physical-design, extended-properties (unrelated)
- dax-studio-workflow (optional for perf review, not basic model review)

---

### Mode C: Extended Properties Generation
**Input**: Schema object (table/column/view/SP) + connection  
**Output**: T-SQL sp_addextendedproperty script  
**Haiku tokens**: ~25K (2 references)

**Core references** (must load):
- extended-properties-templates.md
- data-classification.md (if user asks for sensitivity labels)

**Do NOT load**:
- Any other files (not needed for template-based generation)

---

### Mode D: DAX Measure Review
**Input**: DAX expressions (pasted or from file)  
**Output**: Corrected measures + explanation  
**Haiku tokens**: ~60K (3 references + checklist)

**Core references** (must load):
- sqlbi-dax-patterns.md
- dax-style-guide.md
- dw-review-checklist.md

**Conditional** (load if needed):
- sqlbi-dax-patterns-advanced.md (if user has M2M, TREATAS, paginated reports)
- ssas-tabular-bp.md (if measure naming/display folder questions arise)

**Do NOT load**:
- kimball-patterns (not needed for DAX review)
- elt-patterns, dw-physical-design (unrelated)
- sqlbi-dax-patterns-niche (only if user explicitly requests rare patterns)

---

### Mode E: Bus Matrix Generation
**Input**: Spec (Phase 6 from dw-report-designer) OR existing DW  
**Output**: Markdown bus matrix table  
**Haiku tokens**: ~35K (2 references)

**Core references** (must load):
- kimball-patterns.md (for bus matrix format + conformed dimension concept)

**Conditional** (load if existing DW):
- kimball-advanced-patterns.md (if user has Data Vault or advanced schemas)

**Do NOT load**:
- Other files (not needed for bus matrix synthesis)

---

### Mode F: ELT Pipeline Review
**Input**: SSIS package design OR SP code OR data flow description  
**Output**: Findings report (severity-coded)  
**Haiku tokens**: ~50K (2 references + checklist)

**Core references** (must load):
- elt-patterns.md
- dw-review-checklist.md

**Conditional** (load if needed):
- ssdt-project-structure.md (if SSDT project layout is questionable)
- kimball-patterns.md (if user asks about dimensional conformity in load SPs)

**Do NOT load**:
- DAX patterns, SSAS patterns (not relevant to SSIS/SP review)

---

### Mode G: DevOps Deployment Review
**Input**: Pipeline config OR PS scripts OR deployment approach  
**Output**: Findings report (severity-coded)  
**Haiku tokens**: ~50K (2 references + checklist)

**Core references** (must load):
- devops-operations-patterns.md
- dw-review-checklist.md

**Conditional** (load if needed):
- devops-deployment-patterns.md (if Classic pipeline structure is questionable)
- pbirs-constraints.md (if PBIRS deployment questions arise)

**Do NOT load**:
- Dimensional patterns, DAX patterns (not relevant to deployment)

---

### Mode H: DW Schema Scaffold (Full Build)
**Input**: Confirmed spec from dw-report-designer  
**Output**: SSDT-compatible SQL files + index definitions  
**Haiku tokens**: ~70K (4 references)

**Core references** (must load):
- kimball-patterns.md
- dw-physical-design.md
- elt-patterns.md
- extended-properties-templates.md

**Conditional** (load if needed):
- kimball-advanced-patterns.md (if user has Data Vault or advanced SCD)
- dw-calendar-build.md (if Calendar dimension is new)
- ssdt-project-structure.md (if user asks about file layout)
- cloud-migration-portability.md (if user asks about Tabular → Fabric path)

**Do NOT load**:
- SSAS, DAX patterns (not needed for DW schema)
- elt-patterns is needed for SP naming conventions

---

### Mode I: SSAS Tabular Model Scaffold
**Input**: DW table list + measures list + spec  
**Output**: TMDL files + relationships + display folders  
**Haiku tokens**: ~70K (4 references)

**Core references** (must load):
- ssas-tabular-bp.md
- kimball-patterns.md
- security-implementation.md
- sqlbi-dax-patterns.md (for measure stubs)

**Conditional** (load if needed):
- dw-calendar-build.md (if Calendar dimension is new)
- ssas-deployment-processing.md (if user asks about deployment strategy)
- tabular-editor-2-automation.md (if user asks about BPA or automation)

**Do NOT load**:
- elt-patterns, dw-physical-design (not relevant to SSAS model)

---

### Mode J: Source Stored Procedure Generation
**Input**: Source table DDL + confirmed columns  
**Output**: Staging.Load*, Dimension.Load*, Fact.Load* SPs  
**Haiku tokens**: ~45K (2 references)

**Core references** (must load):
- elt-patterns.md (for SP naming + incremental pattern)
- extended-properties-templates.md (for lineage documentation)

**Conditional** (load if needed):
- kimball-patterns.md (if user asks about dimension conformity in load logic)
- cloud-migration-portability.md (if user asks about cloud-ready SSIS → ADF migration)

**Do NOT load**:
- SSAS, DAX patterns (not needed for SP generation)

---

### Mode K: SSIS Catalog Configuration
**Input**: Source/target servers + SSIS project name  
**Output**: ssis_catalog_configuration.json + documentation  
**Haiku tokens**: ~30K (1 reference)

**Core references** (must load):
- elt-patterns.md (for SSIS project structure + environment variables)

**Do NOT load**:
- Other files (not needed for catalog config)

---

### Mode L: DAX Measure Generation
**Input**: Measure list + spec (types, time intelligence)  
**Output**: TMDL measure definitions OR TE2 script  
**Haiku tokens**: ~60K (3 references)

**Core references** (must load):
- sqlbi-dax-patterns.md
- dax-style-guide.md
- ssas-tabular-bp.md (for display folder naming)

**Conditional** (load if needed):
- sqlbi-dax-patterns-advanced.md (if user has M2M, paginated reports)
- dw-calendar-build.md (if user asks about date table conventions)

**Do NOT load**:
- kimball-patterns, elt-patterns (not needed for measure generation)

---

### Mode M: ADO Classic Pipeline Config Generation
**Input**: Project name + stage names + server list  
**Output**: Task configuration + variable groups  
**Haiku tokens**: ~65K (3 references)

**Core references** (must load):
- devops-deployment-patterns.md
- devops-operations-patterns.md
- elt-patterns.md (for package deployment strategy)

**Conditional** (load if needed):
- ssas-deployment-processing.md (if user asks about SSAS refresh strategy)
- tabular-editor-2-automation.md (if user asks about TE2 automation in pipeline)
- pbirs-constraints.md (if user asks about PBIRS deployment)

**Do NOT load**:
- Dimensional patterns, DAX patterns (not relevant to pipeline deployment)

---

### Mode N: Full DW Scaffold (Orchestrated)
**Input**: Signed-off spec from dw-report-designer  
**Output**: Complete set of Mode H → M artifacts  
**Execution**: Sequential modes with dependency DAG  
**Notes**: Mode N cannot use selective loading (orchestrator manages sub-agent routing; use full context)

---

### Mode O: Physical Design Review
**Input**: Live SQL Server connection OR sys.indexes query output  
**Output**: Findings report (severity-coded)  
**Haiku tokens**: ~40K (2 references)

**Core references** (must load):
- dw-physical-design.md
- dw-review-checklist.md

**Conditional** (load if needed):
- dw-validation-patterns.md (if user asks for validation queries)

**Do NOT load**:
- Dimensional patterns, DAX patterns (not relevant to indexing)

---

### Mode P: Source System Analysis
**Input**: Source database/CSV/manual input  
**Output**: design/entity-map.md (Source Entity Map)  
**Haiku tokens**: ~30K (1 reference)

**Core references** (must load):
- source-system-analysis.md (queries Q1–Q10, classification heuristics, output format)

**Do NOT load**:
- Other files (not needed for source profiling)

---

## Reference Load Sizes (Estimated)

| Mode | Core Refs | Conditional Refs | Haiku Tokens | Truncation Risk |
|---|---|---|---|---|
| A | 3 | 3 | ~50K | ✅ Safe |
| B | 3 | 2 | ~45K | ✅ Safe |
| C | 2 | 0 | ~25K | ✅ Safe |
| D | 3 | 2 | ~60K | ✅ Safe |
| E | 1 | 1 | ~35K | ✅ Safe |
| F | 2 | 2 | ~50K | ✅ Safe |
| G | 2 | 2 | ~50K | ✅ Safe |
| H | 4 | 4 | ~70K | ✅ Safe |
| I | 4 | 3 | ~70K | ✅ Safe |
| J | 2 | 2 | ~45K | ✅ Safe |
| K | 1 | 0 | ~30K | ✅ Safe |
| L | 3 | 2 | ~60K | ✅ Safe |
| M | 3 | 3 | ~65K | ✅ Safe |
| O | 2 | 1 | ~40K | ✅ Safe |
| P | 1 | 0 | ~30K | ✅ Safe |

**Conclusion**: All modes stay under 75K tokens for Haiku 200K window. **No truncation risk.**

---

## Sub-Agent Routing Rules

### When to Route to Haiku Sub-Agent

Route to background `task` agent (Haiku) when:

1. **Mode is explicitly structure-only** (A, B, C, E, F, G, K, O, P) — no semantic reasoning required
2. **User provides single input** (one DDL file, one entity list, one SQL result) — output is deterministic
3. **Output is deterministic** (findings report, entity map, config JSON) — no multi-step confirmation loops needed
4. **References fit in Haiku window** (~70K max; conditional refs loaded only if explicitly needed)

**Do NOT route** (keep on Sonnet orchestrator):
- Mode D (DAX review) — requires nuanced semantic checking; escalation risk if Haiku misses pattern
- Mode H, I, L (full scaffolds) — multiple steps, user sign-off loops, error recovery
- Mode M (pipeline generation) — complexity, user confirmation needed
- Mode N (full build) — orchestrated; Sonnet manages sub-agent routing

---

## Orchestrator Logic (Sonnet)

```
User query → Sonnet infers mode
  ↓
IF mode ∈ {A, B, C, E, F, G, K, O, P}:
  ↓
  Load selective references for mode
  Route to background task (Haiku)
  With explicit checklist from dw-review-checklist.md
  Return results when complete
ELSE (D, H, I, L, M, N):
  ↓
  Process on Sonnet (full context available)
  Use confirmation loops + error recovery as needed
```

---

## Cost Analysis

> ⚠️ **Note**: Cost estimates below are directional only. Actual cost depends on session context size, conversation length, and how many turns reference-heavy modes are invoked. Treat these as order-of-magnitude estimates, not precise billing projections.

### Key Driver: Reference Load Size Per Mode Interaction

When an agent loads references to answer a user query, each reference file adds to the input token count for that turn. The sub-agent pattern reduces this by loading only the references a given mode needs, instead of the full 28-file context.

| Scenario | Avg Input Tokens/Turn | Model | Cost/Turn |
|---|---|---|---|
| Baseline: full context, every turn | ~120K (28 refs avg) | Sonnet $3/1M | ~$0.36 |
| Sub-agent: structure mode (A, B, P) | ~50K (3 refs) | Haiku $1/1M | ~$0.05 |
| Sub-agent: semantic mode (D, H, L) | ~100K (3–5 refs) | Sonnet $3/1M | ~$0.30 |

**Per-turn saving** (structure mode): ~$0.31/turn (86% reduction on those turns)  
**Per-turn saving** (semantic mode vs full context): ~$0.06/turn (17% reduction)

### Why the Previous Math Was Wrong

The prior cost table compared incompatible bases: "$1.37 per project" used `325K tokens × 50 interactions / 1M × $3` (= $48.75, not $1.37). The arithmetic was incorrect and the "40% reduction" claim was false.

**Corrected estimate** (directional):
- A typical project has ~20 structure-mode turns + ~15 semantic turns + ~15 short orchestration turns
- Baseline (Sonnet, full context every turn): 50 × 120K × $3/1M = **$18.00/project**
- Sub-agent pattern: (20 × 50K × $1 + 15 × 100K × $3 + 15 × 40K × $3) / 1M = **$7.30/project**
- **Estimated saving: ~60% per project** (unvalidated — confirm after first real session)

The 60% estimate assumes structure modes dominate, which may not hold for review-heavy projects. **Validate with one real project before relying on this figure.**

---

## Implementation Checklist

- [ ] Create this mapping document (done)
- [ ] Update `ssas-tabular-dw-architect.agent.md` to route Mode B → sub-agent with checklist
- [ ] Update `db-documenter.agent.md` to route applicable modes → sub-agents
- [ ] Test Mode B (Tabular Model Review) with Haiku sub-agent on real session
- [ ] Validate Haiku output quality (findings accuracy, checklist coverage)
- [ ] If test passes, enable for other modes (A, C, E, F, G, K, O, P)

