# CopilotDWTools

A toolkit of GitHub Copilot agents and skills for on-premises SQL Server Data Warehouse,
SSAS Tabular, SSIS ELT, and Power BI Report Server development. Based on Kimball methodology,
SQLBI/DAX Patterns, and Tabular Editor 2 BPA rules.

---

## Repository Structure

```
agents/
  ssas-tabular-dw-architect.agent.md  — Primary review + build agent
  dw-report-designer.agent.md         — Orchestrator: requirements interview → spec → build handoff

skills/
  sql-dw-dimensional-review/
    SKILL.md                           — Core skill entry point (Modes A–N)
    references/
      kimball-patterns.md             — Kimball dimensional modeling core: fact/dim design, SCD types, bus matrix
      kimball-advanced-patterns.md    — Advanced patterns: Data Vault bridging, late-arriving, physical design, snapshot DDL
      sqlbi-dax-patterns.md           — SQLBI/DAX Patterns core: time intelligence, semi-additive, M2M, upstream-first guide
      sqlbi-dax-patterns-advanced.md  — Medium-likelihood DAX patterns: paginated reports, what-if, dynamic security
      sqlbi-dax-patterns-niche.md     — Niche DAX patterns: currency conversion, survey/weighted avg (use rarely)
      ssas-tabular-bp.md              — SSAS Tabular best practices + BPA rules + SSAS schema view contract
      extended-properties-templates.md — sp_addextendedproperty T-SQL templates
      dw-review-checklist.md          — End-to-end DW/SSAS/PBIRS review checklist
      elt-patterns.md                 — SSIS ELT patterns + upstream-first examples + Internal.Lineage DDL
      devops-deployment-patterns.md   — ADO Classic pipeline structure + per-component deployment (DACPAC, SSIS, SSAS, PBIRS)
      devops-operations-patterns.md   — ELT trigger, PS standards, repo structure, PS library, ALM Toolkit, roll-forward incident response
      pbirs-constraints.md            — Power BI Report Server constraints and design rules
      pbix-report-standards.md        — PBIX report standards (Debug tab, table descriptions, measure format)
      data-classification.md          — SQL sensitivity classification and data governance templates
      ssdt-project-structure.md       — SSDT project layout, publish profiles, pre/post-deploy scripts
      ssisdb-catalog-config.md        — SSISDB topology, environment variables, JSON config format
      tabular-editor-2-automation.md  — TE2 CLI flags, C# scripts library, BPA rule JSON
      ssas-deployment-processing.md   — TE2 deploy commands, processing modes, SQL Agent job pattern
      dw-validation-patterns.md       — T-SQL validation queries (orphan facts, unknown members, reconciliation)
      dax-style-guide.md              — DAX coding standard (naming, formatting, VAR/RETURN, upstream-first principle)
      dax-studio-workflow.md          — DAX Studio: Server Timings, VertiPaq Analyzer, benchmarking workflow
      dw-physical-design.md           — Index strategy (CIX/NCI/CCI), staging heap pattern, statistics, partitioning, shared-instance guidance
```

---

## Agents

### DW & SSAS Tabular Architect (`ssas-tabular-dw-architect.agent.md`)

**Model:** Claude Sonnet (with GPT-5.4 self-review gate before delivering any output)

Primary review and build agent. Connects to live SQL Server databases via the MS SQL extension,
reads `.bim`/TMDL files, and works from user-provided DDL. All output is pipeline-executable —
no GUI steps.

**Scope:** SSAS Tabular only — no MDX, no Multidimensional. If a Multidimensional model is
provided, the agent stops and offers a migration path.

Requires: `ms-mssql.mssql` extension for live database connectivity.

---

### DW Report Designer (`dw-report-designer.agent.md`)

**Model:** Claude Opus 4.7 (with GPT-5.5 gap-check run after drafting each interview phase)

Orchestrates new report and DW development from first conversation to signed-off spec, then
hands off to build. Does not generate schemas, TMDL, DAX, or pipelines until the user has
confirmed the complete specification.

**Grain is a hard gate** — the agent cannot proceed to dimensions or measures until the grain
is confirmed and unambiguous.

Coordinates with `ssas-tabular-dw-architect` for source schema validation, then invokes
Modes H–N for artifact generation.

**8 interview phases (in order):**

| Phase | Focus |
|---|---|
| 1 | Business Context |
| 2 | Source Systems |
| 3 | Grain (gate — must be confirmed before continuing) |
| 4 | Measures |
| 5 | Dimensions |
| 6 | Time Intelligence |
| 7 | Sensitivity / Data Classification |
| 8 | Refresh / Scheduling |

---

## Skill Modes (A–N)

### Review Modes

| Mode | Trigger | What it does |
|---|---|---|
| A — DW Schema Review | SQL Server connection / schema DDL | Classifies tables by schema, checks grain/SCD/surrogate keys, audits extended property coverage, flags anti-patterns |
| B — SSAS Tabular Review | .bim / TMDL file or XMLA endpoint | Reviews model against BPA rules; checks naming, relationships, partitions, measures, calculation groups |
| C — Extended Properties | Any DW object | Generates ready-to-run `sp_addextendedproperty` SSDT post-deploy scripts |
| D — DAX Measure Review | Pasted or file-based DAX | Reviews against SQLBI patterns — DIVIDE, VAR, time intelligence, filter context, measure quality checklist |
| E — Bus Matrix | Live schema | Produces enterprise bus matrix from schema |
| F — ELT Review | SSIS packages / source SPs | Reviews against ELT patterns (KingswaySoft, incremental load, lineage, error handling) |
| G — Pipeline Review | ADO Classic pipeline JSON / YAML | Reviews build and release pipeline configs against canonical patterns |

### Build Modes

| Mode | What it generates |
|---|---|
| H — DW Schema Scaffold | `CREATE TABLE` DDL for `Dimension`, `Fact`, `Staging`, `Internal` schemas with org naming conventions |
| I — SSAS TMDL Scaffold | TMDL folder structure (tables, columns, measures, relationships, partitions) from DW schema |
| J — Source SP Generation | Source extraction stored procedures (incremental load pattern with lineage) |
| K — SSIS Catalog Config | `ssis_catalog_configuration.json` with token placeholders for the release pipeline |
| L — DAX Measure Generation | DAX measures following SQLBI patterns with descriptions |
| M — Pipeline Config Generation | ADO Classic build + release pipeline task configurations |
| N — Full Orchestrated Build | Runs H→M in dependency order for a complete new subject area |

---

## How They Work Together

```
User → dw-report-designer (interview, 8 phases)
         ↓ grain gate confirmed
       ssas-tabular-dw-architect (source validation, schema inspection)
         ↓ spec signed off
       SKILL.md Modes H–N (artifact generation)
         ↓
       DW DDL + SSAS TMDL + Source SPs + SSIS Config + DAX + Pipeline Config
```

---

## Org Conventions

| Convention | Rule |
|---|---|
| SSAS engine | Tabular only — no MDX, no Multidimensional |
| Schemas | `Dimension`, `Fact`, `Staging`, `Internal`, `SSAS` (views), `Security`, `Snapshots` |
| Table naming | No table prefixes — schema is the classifier (`Dimension.Customer`, not `DimCustomer`) |
| Surrogate keys | `{Entity}Key` pattern — e.g. `CustomerKey` (not `SK` suffix) |
| Natural keys | `_Source{OriginalName}` prefix — e.g. `_SourceCustomerID` |
| Date dimension | `Dimension.Calendar` / SSAS table `Calendar` |
| Audit / lineage | `Internal.Lineage` + `LineageKey` column — no per-row `ETLBatchID` or `RowCreatedDate` |
| Pipelines | ADO Classic (not YAML). SSIS marketplace task. |
| Tabular Editor | Version 2 (free) at `E:\Tools\TabularEditor\TabularEditor.exe` |
| Reports | Power BI Report Server (on-premises), live connection to SSAS Tabular |
| Security | AD groups only — never individual user accounts. Two standard roles per project: `{Name} Consumers` (Read) and `{Name} Authors` (Read+Process). Same AD groups used for both SSAS role membership and PBIRS folder permissions. |
| Environments | DEV → TEST → UAT → PROD. SUPPORT mirrors PROD (used for production-support investigations). |

## Approved Developer Tools

The following tools are approved for use within this organisation. Agents and generated guidance must only reference these tools — do not suggest alternatives unless the user explicitly asks.

| Tool | Version / Notes |
|---|---|
| **Visual Studio DB Projects** (SSDT) | Database schema, DACPAC build |
| **Git** | Source control (via ADO Server repositories) |
| **Tabular Editor 2.x** | Free; SSAS Tabular model authoring, BPA, deployment |
| **SQL Server Management Studio (SSMS)** | Free; SQL Server and SSAS administration |
| **Power BI Desktop (Report Server edition)** | Must use the Report Server–matched release |
| **DAX Studio** | Free; DAX query profiling and measure authoring |
| **ALM Toolkit** | Free; SSAS model comparison and selective deployment |
| **BIML Express** | Free Visual Studio extension; BIML-based SSIS package generation |
| **Azure DevOps Server** | On-premises; code repos, work items, build/release pipelines |

---

## Installation

### Installing an Agent

Copy the `.agent.md` file to `.github/agents/` in any repository, or install via VS Code URL:

```
vscode:chat-agent/install?url=https://raw.githubusercontent.com/<your-org>/CopilotDWTools/main/agents/ssas-tabular-dw-architect.agent.md
```

### Installing the Skill

```bash
gh skills install <your-org>/CopilotDWTools sql-dw-dimensional-review
```

Or copy `skills/sql-dw-dimensional-review/` to `.github/skills/` in your repo.

### Requirements

- GitHub Copilot Chat (VS Code)
- [MS SQL extension](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql) — required for live database connectivity
- [Tabular Editor 2](https://github.com/TabularEditor/TabularEditor/releases) — required for SSAS model work (free)

---

## References

- *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
- [daxpatterns.com](https://www.daxpatterns.com) — SQLBI
- [SQLBI.com](https://www.sqlbi.com) — DAX and Tabular best practices
- [Tabular Editor BPA Rules](https://github.com/TabularEditor/BestPracticeRules) — community best practice rules
- [Tabular Editor Community Scripts](https://github.com/TabularEditor/Scripts)
- [Microsoft SSAS Tabular docs](https://learn.microsoft.com/en-us/analysis-services/tabular-models/tabular-models-ssas)
