# DW/BI Decisions Register — [Project Name]

> **How to use this file**
> Load this file at the start of any new agent conversation about this project. The `DW Report Designer` agent will read it, confirm whether answers are still current, and skip re-asking questions that are already documented here.
>
> Developers can also read this file to understand *why* the model was designed the way it was — it records the business rationale, not just the technical choices.
>
> Commit this file to source control alongside the project code.

**Project**: [Project Name]
**Subject area**: [e.g. Finance / HR / Operations / Projects]
**Schema version**: 1.0
**Status**: DRAFT *(set to CONFIRMED after spec sign-off)*
**Created**: [YYYY-MM-DD]
**Last updated**: [YYYY-MM-DD]
**Last confirmed**: [YYYY-MM-DD — date a user walked through and verified these answers]
**Spec file**: [relative path to the linked design spec, once it exists]
**Bus matrix**: [confirmed in spec on YYYY-MM-DD / not yet confirmed]

---

> **Scope markers**: 🏢 = assumed org-wide standard | 📋 = project-specific | ❓ = uncertain, confirm with broader team before applying to other projects
>
> **Confidence**: `confirmed` = user explicitly stated | `assumed` = agent inferred from context | `uncertain` = user expressed doubt or answer was ambiguous

---

## Reporting Intent

| # | Question | Answer | Confidence | Notes |
|---|---|---|---|---|
| RI-01 | Primary business questions this report must answer | | | |
| RI-02 | Primary users (role / team) | | | |
| RI-03 | Decision this report drives | | confirmed | |
| RI-04 | Action taken after the decision is made | | confirmed | |
| RI-05 | How often the decision is made (cadence) | | confirmed | |
| RI-06 | Acceptable data staleness for this decision | | confirmed | |

---

## Business Definitions

| # | Decision | Answer | Scope | Confidence | Rationale | Last confirmed |
|---|---|---|---|---|---|---|
| BD-01 | NULL / blank measures — shown as BLANK() or zero? | | 🏢 | | | |
| BD-02 | Unlinked FK records — mapped to "Unknown" row or excluded? | | 🏢 | | | |
| BD-03 | "Open" status — source system values that mean open/active | | 📋 | | | |
| BD-04 | "Closed" status — source system values that mean closed/complete | | 📋 | | | |
| BD-05 | Record corrections — use original, corrected, or both versions? | | 🏢 | | | |
| BD-06 | Default exclusions — records always excluded from the DW (test, void, internal) | | 📋 | | | |
| BD-07 | Default active/all filter — active records only, all records, or user-toggleable? | | 📋 | | | |
| BD-08 | Counting semantics — DISTINCTCOUNT(entity) or COUNTROWS(fact)? | | 📋 | | | |
| BD-09 | Negative values / reversals — net against positive values or surface separately? | | 🏢 | | | |
| BD-10 | Positive variance sign — over-budget (bad) or above-target (good)? | | 📋 | | | |
| BD-11 | Reporting currency | | 🏢 | | | |
| BD-12 | Rounding precision (decimal places; row-level or aggregate-level) | | 📋 | | | |
| BD-13 | Period close window — how many days after period end can data still arrive? | | 🏢 | | | |
| BD-14 | Point-in-time aggregation — end-of-period, start-of-period, or average? | | 📋 | | | |

---

## Date & Time Conventions

| # | Decision | Answer | Scope | Confidence | Rationale | Last confirmed |
|---|---|---|---|---|---|---|
| DT-01 | Date range boundary (inclusive-inclusive / half-open) | | 🏢 | | | |
| DT-02 | Fiscal year start month | | 🏢 | | | |
| DT-03 | Working day definition (weekdays only / exclude public holidays) | | 🏢 | | | |
| DT-04 | Holiday calendar (BC provincial / federal Canadian / custom) | | 🏢 | | | |

---

## Data Architecture Assumptions

*Populated during Phases 6–8 of the interview. These affect build mode choices and must be confirmed before Mode N runs.*

| # | Assumption | Answer | Scope | Confidence | Rationale | Last confirmed |
|---|---|---|---|---|---|---|
| DA-01 | Default SCD type for slowly changing dimensions | | 🏢 | | | |
| DA-02 | History required — as-was at transaction time, or always-current? | | 📋 | | | |
| DA-03 | Authoritative source when multiple sources contain the same entity | | 📋 | | | |
| DA-04 | Row-level security (RLS) required? If yes: filtering column | | 📋 | | | |

---

## Deferred Scope

*Items identified during requirements gathering that are out of scope for this delivery. Generated from the optional deferred reports step after bus matrix sign-off. These are natural extensions of the confirmed dimensional model.*

| # | Report / Feature | Why Valuable | Why Deferred | Dependency / Trigger | Suggested On |
|---|---|---|---|---|---|
| DS-01 | | | | | |

---

## Change Log

| Date | Changed by | What changed |
|---|---|---|
| [YYYY-MM-DD] | DW Report Designer agent | Initial draft created from interview |
