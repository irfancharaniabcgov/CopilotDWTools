# Follow-Up Work: Model Assignment Implementation

**Status**: Model assignment committed (e9f7744). Phase 1 ready to validate.

---

## Blocking Tasks

### 1. Identify Nano-Eligible Tasks (TBD)

You mentioned nano ($0.20/1M) could work for trivial tasks with proper orchestration.

**Required before nano exploration**:
- Identify 2–3 specific tasks that are:
  - Non-blocking (failure doesn't cascade)
  - Structured input/output (JSON/YAML)
  - Low reasoning (formatting, templating, syntax checks only)
  
**Examples** (to be confirmed):
- Format session status message (e.g., "Phase 1 ✅; Phase 2 🔄 in progress; 1 ambiguity flagged")
- Substitute parameters into a template (no inference)
- Validate JSON/YAML syntax (not semantics)

**Action**: Define tasks + submit as work item for nano pilot.

---

### 2. Cache Strategy Implementation (Phase 1 Q1–Q3)

**Why**: Phase 1 questions (fiscal year, refresh cadence, consumers) repeat across interviews. Caching these can save 5–10%.

**Required**:
- Identify static reference sections that can be cached (e.g., `kimball-patterns.md` definitions, SQLBI pattern examples)
- Prepare cache-friendly agent context (separate reusable context from interview-specific context)
- Set up cache headers in Phase 1 prompt

**Estimated savings**: 5–10% on Phase 1 repeat interviews.

**Timeline**: After Phase 1 validation (once contradiction detection ≥90%).

---

## Validation Gates (Do These BEFORE Full Rollout)

### Phase 1 Real Interview Test

**Run**: 1 interview using new Sonnet assignment (dw-report-designer Phase 1)

**Metrics to collect**:

1. **Contradiction detection rate**: Did agent catch user saying different things?
   - Examples: "Daily refresh" then "monthly reporting"
   - Measurement: Count contradictions agent flagged / total contradictions in transcript
   - Pass threshold: ≥90%

2. **Clarification quality**: When user answer was ambiguous, did agent ask for clarification (not assume)?
   - Measurement: Count clarifications asked / count ambiguous answers
   - Pass threshold: ≥1 clarification per 3 ambiguous answers

3. **User satisfaction**: Did user feel interview was thorough and fair?
   - Measurement: Ask user (1–5 scale)
   - Pass threshold: ≥4/5

4. **Token usage** (if available): Measure actual cost vs estimate
   - Baseline (Haiku estimate): ~50K tokens × $1.00/1M = $0.05
   - Sonnet actual: Compare
   - Target: Confirm <$0.20

### Decision Rule

✅ **PASS** (keep Sonnet Phase 1):
- Contradiction detection ≥90%
- Clarification rate ≥1 per 3 ambiguous
- User satisfaction ≥4/5

❌ **FAIL** (revert to Opus):
- If any metric drops below threshold, revert Phase 1 model to Claude Opus 4.7
- Escalate to planning for quality improvement

---

## Post-Launch Monitoring (Ongoing)

Once agents are live with new assignments:

### Weekly (First 4 weeks)
- [ ] Run 1–2 Phase 1 interviews with Sonnet
- [ ] Collect metrics (contradiction, clarification, satisfaction)
- [ ] Log metrics in `decisions/validation-results.md` (create as tracking file)

### Monthly (After 4 weeks)
- [ ] Aggregate metrics across 5+ interviews
- [ ] If contradiction detection <90%, escalate
- [ ] If cost > estimate by >20%, investigate token usage

### Quarterly
- [ ] Review cost savings vs estimate
- [ ] Revisit Haiku Phase 1 (if Sonnet quality consistently >95%, pilot Haiku on 10% of interviews)
- [ ] Assess cache strategy impact

---

## Files Created This Session

| File | Purpose | Status |
|---|---|---|
| `decisions/model-selection-framework.md` | Decision framework: metrics, thresholds, rationale | ✅ Reference doc |
| `decisions/model-assignment-final.md` | Final model assignments + cost/risk summary | ✅ Reference doc |
| `decisions/validation-results.md` | TBD — tracking file for post-launch metrics | Create when first interview runs |

---

## Rollout Checklist

- [x] Model assignments updated (dw-report-designer Phase 1: Sonnet; ssas-architect: Haiku + Mode D GPT-5.4)
- [x] Decision framework documented
- [x] Changes committed (e9f7744)
- [ ] **Run Phase 1 validation interview** (before full rollout) ← DO THIS NEXT
- [ ] Collect metrics
- [ ] If pass: approve rollout; if fail: escalate
- [ ] Set up weekly monitoring
- [ ] Identify nano tasks (submit as separate work item)
- [ ] Plan cache strategy (Phase 2)

---

## Summary: What Changed

✅ **dw-report-designer**
- Phase 1: Haiku 4.5 → **Claude Sonnet 4.6**
- Rationale: Quality critical for interviews; contradiction detection must be >90%
- Cost: Slightly higher per interview, but prevents rework (rework >> token cost)

✅ **ssas-tabular-dw-architect**
- Default: GPT-5.4 → **Claude Haiku 4.5**
- Mode D: Still GPT-5.4 (pattern matching, not semantic reasoning)
- Rationale: 80% of work is structure-only (checklist validation); Haiku sufficient
- Savings: ~60% vs previous all-GPT-5.4

✅ **db-documenter**
- No change (already Sonnet 4.6, appropriate for inference)

**Estimated annual savings**: 15% ($7.5k → $6.4k) assuming 100 projects/year.

**Risk**: Phase 1 quality drops below 90% → revert to Opus (fallback cost: ~$90/project, net loss). **Validation gate prevents this.**

---

## Next User Action

1. **Validate Phase 1** with one real interview (collect metrics)
2. **If pass**: Approve full rollout
3. **If fail**: Escalate + investigate (likely misalignment of agent context or user expectation)
4. **Identify nano tasks** (async, submit separately)
5. **Plan cache strategy** (Phase 2)

