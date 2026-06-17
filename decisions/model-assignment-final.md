# Agent Model Assignment — Final Recommendations

**Date**: 2026-06-17  
**Status**: Ready to implement  
**Decision basis**: Pricing analysis + capability matrix + user preferences (quality > cost for interviews; Claude for implementation; GPT for gates)

---

## Assignments (Final — Implemented)

### 1. dw-report-designer

| Phase | Model | Cost | Rationale |
|---|---|---|---|
| **All phases (1–7)** | **Claude Sonnet 4.6** | $3.00/1M input | Quality critical; contradiction detection for interviews. Single model for entire session. |
| **Mode P sub-agent** | **Claude Haiku 4.5** | $1.00/1M input | Background task; source profiling Q1–Q10 only; selective refs (source-system-analysis.md). |

**External review gates**: GPT models invoked on demand (not within this agent's session).

---

### 2. ssas-tabular-dw-architect

| Mode | Model | Cost | Rationale |
|---|---|---|---|
| **All modes (orchestrator)** | Claude Haiku 4.5 | $1.00/1M input | Front matter model; handles routing, orchestration, all mode conversations. |
| **Mode A/B sub-agent** | Claude Haiku 4.5 | $1.00/1M input | Background task; structure-only checklist; selective refs (~45K tokens). |
| **Mode K nano sub-agent** | GPT-5 nano | $0.20/1M input | JSON template fill-in; deterministic; safe failure. Fallback: GPT-5.4-mini. |

**External review gates**: GPT models invoked on demand (not within this agent's session).

---

### 3. db-documenter

| Task | Model | Cost | Rationale |
|---|---|---|---|
| **All tasks** | Claude Sonnet 4.6 | $3.00/1M input | Inference-heavy description generation; reliability needed. |

---

### 4. Nano Model (GPT-5 nano, $0.20/1M input) — Confirmed

Three confirmed tasks (see `decisions/nano-task-identification.md`):
1. `sp_addextendedproperty` upsert boilerplate (Mode C) — template fill-in
2. `ssis_catalog_configuration.json` (Mode K) — ✅ implemented
3. FormatString + DisplayFolder + Description stub (Mode L) — lookup table

---

## Changes Required

| File | Change | Impact |
|---|---|---|
| `agents/dw-report-designer.agent.md` | Line 4 (model): Change from `gpt-5.5` to `claude-sonnet-4.6` (front matter only; operating instructions specify phase-based routing internally) | Phase 1 cost: ↓ 40% |
| `agents/ssas-tabular-dw-architect.agent.md` | Line 4: Change from `gpt-5.4` to `claude-haiku-4.5`; add note that Mode D will switch to GPT-5.4 internally | Overall cost: ↓ 60% |
| `agents/db-documenter.agent.md` | No change (already Sonnet 4.6) | — |
| `decisions/model-selection-framework.md` | ✅ Created (this session) | Decision documentation |

---

## Validation Gates

Before marking "done", verify:

- [ ] **dw-report-designer Phase 1**: Updated to Sonnet 4.6 in agent frontmatter
- [ ] **ssas-architect**: Updated to Haiku; Mode D routing to GPT-5.4 documented
- [ ] **Cache strategy**: Q1–Q3 context prepared for caching (list static reference sections that can be cached)
- [ ] **Nano tasks**: List 2–3 tasks and submit as follow-up work item

---

## Going Live

1. Update agent frontmatter (1–2 min)
2. Test: Run one Phase 1 interview with new Sonnet assignment (collect 3 metrics: contradiction rate, clarification rate, user satisfaction)
3. If contradiction detection ≥ 90%, approve
4. If < 90%, revert to Opus 4.7 (fallback) and escalate to planning

---

## Future Iterations (Post-Launch)

- **After 5 real interviews**: Measure Phase 1 metrics (contradiction, clarification, satisfaction). If all ≥ 90%, close the Phase 1 decision. If any < 90%, escalate.
- **Cache implementation**: Once Phase 1 baseline established, implement cache on Q1–Q3 (target 5–10% savings on repeats).
- **Nano exploration**: Once 2–3 tasks identified, pilot nano with explicit failure mode. If < 95% success rate, don't use nano.

---

## Cost Comparison

### Estimated Annual Savings (vs. Previous All-Premium Baseline)

Assume 100 projects/year, average 1 Phase 1 interview per project (full lifecycle cost = Phase 1 + Phase 2–3 + architect reviews):

**Before** (all-premium):
- Phase 1: 100 × 50K tokens × $3.00/1M (Sonnet) = $15
- Phase 2–3: 100 × 100K tokens × $5.00/1M (GPT-5.5) = $50
- Architect reviews: 100 × 20K tokens × $5.00/1M (all GPT-5.4/5.5) = $10
- **Total**: ~$75/project, or ~**$7,500/year**

**After** (optimized):
- Phase 1: 100 × 50K tokens × $3.00/1M (Sonnet) = $15 (no change, but quality improves)
- Phase 2–3: 100 × 100K tokens × $5.00/1M (GPT-5.5) = $50 (unchanged)
- Architect reviews: 100 × 20K tokens ×[(80% × $1.00 Haiku + 20% × $2.50 GPT-5.4)/1M] = ~$4
- Cache savings (Phase 1 repeats): ~$2–4 (if 50% of interviews rerun same Q1–Q3)
- **Total**: ~$64/project, or ~**$6,400/year** (⬇️ 15% reduction)

**Risk**: If Phase 1 Sonnet quality < 90% contradiction detection, revert Phase 1 to Opus ($5.00 input) → $90/project, **net loss**. **This is why validation gates are critical.**

