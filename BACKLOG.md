# Toolkit Development Backlog

> Items deferred during development sessions. Review periodically and promote to active work when ready.

## Research Spikes (not started)

| Spike | Time-box | File | Notes |
|---|---|---|---|
| PBIP on-prem viability | 4h | `design/spikes/pbip-on-prem-viability.md` | Can PBIP format be used with PBIRS-optimized Desktop? |
| Automated regression testing (xUnit + ADOMD.NET) | 6h | `design/spikes/automated-regression-testing.md` | DAX measure + SP validation; reviewed by Opus 4.7 + GPT-5.5 |

## Periodic Maintenance

| Task | Trigger | Owner |
|---|---|---|
| Review `pbirs-constraints.md` feature table | Each PBIRS release (~Jan, May, Sep) or plugin version bump | Toolkit maintainer |
| Confirm user's PBIRS version | Quarterly on long engagements | Agent (automated prompt) |
| Bump plugin version (`plugin.json`) | New capabilities added, behavioural changes, or review fixes applied | Toolkit maintainer |
| Review model tier guidance in all 3 agents | When a major new model tier is released (new Claude/GPT generation) — confirm lightweight/mid-tier/premium classifications still reflect current model landscape | Toolkit maintainer |

## Future Enhancements (parked)

| Item | Priority | Context |
|---|---|---|
| CHANGELOG.md | Low | Declined for now; reconsider if plugin gains external users |
| Warm/cold cache warming — practical SQL Agent job pattern | Low | One-liner exists in `performance-end-to-end.md`; expand only if user asks |
| RLS/role-specific performance testing in regression suite | Medium | Noted in spike Phase 2; depends on regression testing spike adoption |
| Columnstore maintenance scheduling guidance | Low | Eliminated — DBA-managed; existing health query is sufficient |
| SSAS thread pool / CoordinatorExecutionMode | Low | Eliminated — DBA-only; marked as review suggestion in `pbirs-constraints.md` |
| Report-level visual regression testing | Low | No viable automation for PBIRS live-connection reports; revisit if tooling emerges |
| TMDL-driven test stub auto-generation | Medium | R5 in regression testing spike; depends on spike outcome |
| Paginated report (Report Builder) design patterns | Medium | Agent identifies when RDL is needed; no detailed RDL design guidance exists yet |
| DAX UDF guidance (when PBIRS supports CL 1702+) | Blocked | Cloud-only today; revisit when PBIRS gains support |
| Visual Calculations guidance | Blocked | Cloud-only today; revisit when PBIRS gains support |
