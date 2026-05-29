# Copilot DW Tools

A GitHub Copilot agent and skill set for reviewing SQL Server Data Warehouses, 
Analysis Services Tabular models, and DAX measures using Kimball methodology
and SQLBI/DAX Patterns best practices.

## Contents

```
agents/
  ssas-tabular-dw-architect.agent.md   ← Custom Copilot agent

skills/
  sql-dw-dimensional-review/
    SKILL.md                            ← Skill entry point
    references/
      kimball-patterns.md              ← Kimball methodology reference
      sqlbi-dax-patterns.md            ← SQLBI/DAX Patterns reference
      ssas-tabular-bp.md               ← SSAS Tabular best practices + DMV queries
      extended-properties-templates.md ← sp_addextendedproperty T-SQL templates
      dw-review-checklist.md           ← End-to-end review checklist
```

## Installing the Agent

Download `agents/ssas-tabular-dw-architect.agent.md` and place it in `.github/agents/`
in any repository where you want to use it.

Or install via URL in VS Code:
```
vscode:chat-agent/install?url=https://raw.githubusercontent.com/<your-org>/CopilotDWTools/main/agents/ssas-tabular-dw-architect.agent.md
```

## Installing the Skill

```bash
gh skills install <your-org>/CopilotDWTools sql-dw-dimensional-review
```

Or manually copy the `skills/sql-dw-dimensional-review/` folder to `.github/skills/` in your repo.

## What It Does

The **ssas-tabular-dw-architect** agent supports 5 review modes:

| Mode | Description |
|---|---|
| **DW Schema Review** | Classifies tables, checks grain/SCD/surrogate keys, audits extended property coverage |
| **SSAS Tabular Review** | Reviews .bim / TMDL files or live DMV queries for naming, relationships, measures, partitions |
| **Extended Properties** | Generates ready-to-run `sp_addextendedproperty` scripts for any DW object |
| **DAX Measure Review** | Reviews pasted or file-based DAX against SQLBI patterns |
| **Bus Matrix Generation** | Produces an enterprise bus matrix from a live schema |

## Requirements

- [MS SQL extension for VS Code](https://marketplace.visualstudio.com/items?itemName=ms-mssql.mssql) (`ms-mssql.mssql`) for live database connectivity
- GitHub Copilot Chat

## References

The bundled reference files are based on:
- *The Data Warehouse Toolkit, 3rd Edition* — Ralph Kimball & Margy Ross
- [daxpatterns.com](https://www.daxpatterns.com) — Marco Russo & Alberto Ferrari (SQLBI)
- [SQLBI.com](https://www.sqlbi.com) — DAX and Tabular best practices
- [Microsoft SSAS Tabular documentation](https://learn.microsoft.com/en-us/analysis-services/tabular-models/tabular-models-ssas)
