# Model Selection Framework — Token Efficiency with Quality Guard

**Decision date**: 2026-06-17  
**User priority**: Quality > Cost (especially interviews); balance both elsewhere  
**Vendor preference**: Claude for implementation, GPT for review gates

---

## 1. Phase 1 (Basic Interview: Q1–Q3)

### Candidates
| Model | Cost/1M input | Context | Strength | Risk |
|---|---|---|---|---|
| **Haiku 4.5** | $1.00 | 200K | **Cheap, fast** | May miss nuance; truncates if context > 200K |
| **Sonnet 4.6** | $3.00 | 200K | **Balanced**; catches contradictions | 3x cost vs Haiku |
| Claude Opus 4.7 | $5.00 | 200K | Strongest reasoning | Overkill; 5x Haiku cost |

### Questions to Test (Q1–Q3)
1. **"What's your fiscal year pattern?"** — Simple structured Q (Calendar start/end)
2. **"How often is data refreshed?"** — Semi-open Q (daily/hourly/yearly, tied to grain)
3. **"Who are the report consumers?"** — Open-ended Q (roles, data literacy, distribution format)

### Quality Metrics

| Metric | Measurement | Pass Threshold | Why It Matters |
|---|---|---|---|
| **Contradiction detection** | Does agent catch user saying different things in same session? (e.g., "daily refresh" then "monthly reporting") | **>90% catch rate** | If interviewer misses contradictions, spec is broken at sign-off |
| **Clarification quality** | When user answer is ambiguous, does agent ask for clarification (not assume)? | **Agent asks ≥1 clarifying Q per 3 ambiguous answers** | Haiku may skip clarification to save tokens; Sonnet more thorough |
| **Spec completeness** | After 3 Q, can interviewer move to Phase 2 or do they need re-ask? | **≥90% of users move forward without re-ask** | Rework = hidden cost |
| **Reference integrity** | Does agent correctly cite reference (e.g., "SCD Type 1 keeps things simple") when explaining guidance? | **100% accuracy on cited patterns** | Hallucinated references erode trust |

### Decision Rule

✅ **DECISION**: Use **Claude Sonnet 4.6** for Phase 1.

**Rationale**:
- Haiku saves 66% cost but risks spec ambiguities that cause rework downstream (rework cost >> 66% savings)
- User priority is quality for interviews; Sonnet's contradiction detection + clarification is worth 3x cost
- Sonnet matches user vendor preference (Claude for implementation)
- If Phase 1 proves >95% contradiction-free on Sonnet, revisit Haiku in future iteration

**Mitigation**: Implement cache on Q1–Q3 (these repeat across interviews) → save 5–10% even on Sonnet.

---

## 2. Mode B/D (SSAS Tabular Semantic Review)

### Clarification Needed: What Is "Semantic Review"?

**Two interpretations** of ssas-architect Mode B/D task:

| Task | Description | Tool | Cost/1M input | When |
|---|---|---|---|---|
| **Structure-only review** | Does the DAX measure follow SQLBI pattern structure? (e.g., uses DIVIDE, VAR/RETURN, proper hierarchy) | GPT-5.4 ("Versatile") | $2.50 | Checklist-driven; pattern matching |
| **True semantic review** | Does the measure compute the **correct business logic**? (e.g., "revenue per customer at month-end, excluding returns") | GPT-5.5 ("Powerful") | $5.00 | Requires reasoning about domain intent |

**Current agent description** (from your existing agent files) suggests **Mode B = schema validation**, **Mode D = DAX review**.

If Mode D is **structure-only** (checklist against `sqlbi-dax-patterns.md`):
- Use **GPT-5.4** ($2.50) ✅ — saves 50% vs GPT-5.5
- Risk: low (pattern matching is reliable)

If Mode D is **semantic** (does measure mean the right thing?):
- Use **GPT-5.5** ($5.00) 🔴 — required for reasoning
- Risk: high if you use GPT-5.4 (may miss logic errors)

### Decision Rule (Conditional on Your Clarification)

**I recommend**: Ask yourself—when reviewing a DAX measure, do you ask:
- **A** (structure-only): "Does it use DIVIDE? Does it have VAR? Is it formatted?" → **Use GPT-5.4**
- **B** (semantic): "Does this measure actually compute what the business needs?" → **Use GPT-5.5**

If mostly **A**, downgrade to GPT-5.4 and save 50%.  
If mostly **B** or mixed, stay with GPT-5.5.

**Until you clarify**, I'll assume **mixed** → **keep GPT-5.5**.

---

## 3. Nano Model Exploration (GPT-5 nano, $0.20)

### When to Use

Only for **trivial, non-blocking tasks** where failure is contained:
- Status message formatting (no logic)
- Template parameter substitution (no inference)
- Simple validation of syntax (not semantics)

### Handoff Rule (You Mentioned This)

Nano agent must:
1. **Receive full context** from orchestrating agent (don't assume prior state)
2. **Fail explicitly** — if uncertainty > 20%, escalate to higher model (not silently guess)
3. **Produce structured output** (JSON/YAML) so orchestrator can validate before proceeding

### Example: Phase 1 Status Update

Orchestrating agent sends nano:
```
Task: Summarize interview results

Input:
  - Phase: 1
  - Questions answered: Q1 (Fiscal year), Q2 (Refresh cadence), Q3 (Consumers)
  - Ambiguities flagged: 1 (refresh timing needs clarification in Phase 2)
  - Contradictions: 0
  - User approved: yes/no

Output: 1-sentence status (e.g., "Phase 1 ✅ Complete; 1 follow-up needed in Phase 2")
```

If nano produces nonsensical output, orchestrator catches it and re-runs with Sonnet.

### Decision Rule

✅ **DECISION**: Explore nano for **trivial status/formatting tasks** only.

**Prerequisite**: Define 2–3 specific nano-eligible tasks first.  
**Mitigation**: Nano must fail explicitly (not silently); orchestrator validates output.

---

## 4. Implementation Timeline

### Phase 2A (Model Assignment — THIS SESSION)

- [x] Analyze pricing + capabilities
- [x] Create decision framework (this document)
- [ ] **Clarify Mode D task** (structure-only vs semantic?)
- [ ] Identify 2–3 nano-eligible tasks
- [ ] Update agent frontmatter with final model assignments

### Phase 2B (Validation — FUTURE SESSIONS)

Once agents are live:
- Monitor Phase 1 interviews for contradiction detection rate (target: >90%)
- Track clarification quality (target: ≥1 clarifying Q per 3 ambiguous answers)
- Measure token usage (cache impact on Q1–Q3 repeats)
- If contradiction detection drops below 90%, revert Phase 1 to Opus

---

## 5. What to Look For Going Forward

### Metric Collection Template

When you run interviews, track:

```
Session: [date]
Agent: dw-report-designer Phase 1
Model: [Sonnet/Haiku/etc]

Phase 1 Results:
  Q1 (Fiscal Year):
    - User answer: [recorded]
    - Agent clarified? [yes/no]
    - Contradiction found? [yes/no]
    - Moved to Phase 2? [yes/no]
  Q2 (Refresh Cadence):
    - User answer: [recorded]
    - Agent clarified? [yes/no]
    - Contradiction found? [yes/no]
  Q3 (Consumers):
    - User answer: [recorded]
    - Agent clarified? [yes/no]
    - Contradiction found? [yes/no]

Overall:
  - Clarifications asked: [count]
  - Contradictions caught: [count]
  - User satisfied with answers? [yes/no]
  - Token usage estimate: [if known]
```

**Decision trigger**: If contradiction detection < 90%, escalate model to Opus.

---

## 6. Pending Decisions

| Item | Status | Action |
|---|---|---|
| Mode D task type (structure vs semantic) | 🔴 BLOCKED | User to clarify |
| Nano-eligible task list | 🔴 BLOCKED | Identify 2–3 tasks |
| Cache strategy implementation | 🟡 PENDING | Implement after Phase 1 finalized |
| Haiku Phase 1 revisit | 🔵 FUTURE | Revisit if Sonnet tracks >95% contradiction detection |

---

## Summary: Current Recommendations

| Agent | Phase/Mode | Model | Cost | Rationale |
|---|---|---|---|---|
| dw-report-designer | Phase 1 | Sonnet 4.6 | $3.00/1M | Quality > cost; contradiction detection critical |
| dw-report-designer | Phase 2–3 | GPT-5.5 | $5.00/1M | Complex spec validation; edge cases |
| ssas-architect | Mode A/M (structure) | Haiku 4.5 | $1.00/1M | Checklist-only, no reasoning needed |
| ssas-architect | Mode B/D | ⏳ GPT-5.4 or GPT-5.5? | $2.50–5.00/1M | Depends on semantic vs structure task |
| db-documenter | Q-query generation | Sonnet 4.6 | $3.00/1M | Inference-heavy; needs reliability |
| Trivial tasks | Status/format | Nano 4.5 | $0.20/1M | IF explicit failure mode + validation |

**Estimated savings vs current**: 15–20% (Phase 2 async + Haiku structure tasks + cache).

