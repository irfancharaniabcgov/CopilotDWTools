# Spike: Automated Regression Testing for DW/SSAS Stack

| Field | Value |
|---|---|
| **Status** | Not started |
| **Time-box** | 6 hours (Phase 1 PoC) + follow-up hardening |
| **Owner** | TBD |
| **Created** | 2026-06-08 |
| **Reviewed by** | Claude Opus 4.7, GPT-5.5 (2026-06-08) |

## Problem Statement

No automated regression testing exists for the DW/SSAS/DAX stack. Changes to stored procedures, model structure, or DAX measures can introduce silent regressions that are only caught during manual testing or (worse) in production. Manual testing is tedious, time-consuming, and coverage is inconsistent.

## Hypothesis

Use **xUnit as the common test runner** with ADOMD.NET for DAX/SSAS tests and SqlClient for DW stored procedure tests. Combined with existing TE2 BPA (model structure), this avoids introducing tSQLt or NBi.

**Why not tSQLt:** Single-language preference (.NET/C#); avoids CLR enablement requirement on DW server; xUnit + SqlClient can validate SP outputs directly. Trade-off acknowledged: tSQLt's `FakeTable` isolation is stronger for complex MERGE logic — accepted, mitigated by dedicated test database strategy.

**Why not NBi:** Single maintainer, last release Aug 2023, low confidence for long-term investment.

## Constraints

- Must run in ADO Classic pipelines (self-hosted agents, Windows, x64)
- Must use tools already available or easily added (.NET, xUnit, ADOMD.NET NuGet, SqlClient)
- TE2 BPA already runs in CI/CD — do not duplicate its scope
- No cloud dependencies (on-prem SQL Server 2022, SSAS Tabular)
- Connection strings supplied via ADO variable groups (no secrets in code)

## Key Decisions (resolve before coding)

| Decision | Options | Notes |
|---|---|---|
| **Test model strategy** | Dedicated CI SSAS database (frozen snapshot) vs process-on-demand | Frozen = fast, deterministic; on-demand = realistic but slower |
| **Test DW strategy** | Dedicated CI database (DACPAC + fixture seed) vs shared DEV | Dedicated = isolated; shared = risky for parallel runs |
| **Golden result format** | Inline constants vs `.expected.json` files | JSON is reviewable in PR; constants are simpler for PoC |
| **Numeric tolerance** | `Math.Abs(actual - expected) < 0.01` for currency/decimal | Avoids false failures from aggregation order |

## Research Questions

### R0: Connectivity & Auth (validate first)
- Which NuGet package? (`Microsoft.AnalysisServices.AdomdClient.retail.amd64` vs `.NetCore` variant)
- Does the ADO agent service account have SSAS read permission on the test model?
- Does ADOMD.NET connect successfully with Windows auth from the build agent?
- Is `Provider=MSOLAP` needed or does native ADOMD suffice?
- **Measure**: cold connection time (target < 5s)

### R1: xUnit + ADOMD.NET for DAX Measure Regression
- Can xUnit connect to SSAS via ADOMD.NET and execute DAX queries?
- Use `EVALUATE` + `AdomdDataReader` pattern (not `CellSet`) for all tests
- Golden-result comparison: pin date context with explicit `CALCULATETABLE(..., 'Calendar'[Year] = 2024)` for time-intelligence measures
- Tolerance rules for decimal/currency comparisons
- Use `IClassFixture<T>` to share ADOMD connection across test class (avoid per-test connection overhead)
- **PoC must include**: at least one time-intelligence measure (e.g., Sales YTD) with pinned date context

### R2: xUnit for DW Stored Procedure Validation
- Can xUnit execute SPs via `Microsoft.Data.SqlClient` and assert row counts / output values?
- Transaction rollback acceptable only for simple validation queries
- For load SP validation: dedicated test database + truncate/reseed between test classes
- Go deeper than row counts — validate at least one each of:
  - Staging duplicate natural key detection
  - Dimension unknown member / current-row check
  - Fact orphan FK check
  - Lineage success check (`Internal.Lineage` row created)

### R3: Pipeline Integration
- Test step placement: **after** DB deploy → SSIS deploy → SSAS deploy → ELT run → SSAS process → readiness check → **then test** → gate promotion
- Gate UAT promotion on test pass (fail = block)
- Test execution time budget: < 5 minutes for the full suite
- Use `dotnet test` task with TRX output for ADO test result publishing
- Disable xUnit parallelization for SSAS integration tests (`[CollectionDefinition]`)

### R4: Test Data Strategy
- Seed data: static fixtures (deterministic, reviewable)
- Isolate from production data drift via dedicated CI database
- Lightweight SSAS model (subset of tables) for faster processing
- SSAS readiness gate: query `$SYSTEM.TMSCHEMA_PARTITIONS` — all partitions must have `State = 1` (Ready) before DAX tests execute

### R5: TMDL-Driven Test Generation (future)
- Is `[Theory]` with `MemberData` reading measure names from TMDL feasible?
- Can TE2 TOM / TMDL parsing produce a list of measures + expected patterns?
- Does this produce useful test failure messages (measure name in output)?
- 15-minute feasibility check during PoC

## Evaluation Criteria

| Criterion | Acceptable | Ideal |
|---|---|---|
| Tooling | xUnit + ADOMD.NET + SqlClient only | Same |
| Pipeline fit | Runs as ADO Classic task | Gate for UAT promotion |
| Coverage | Top 10 measures + top 5 SPs | All measures + all load SPs |
| Execution time | < 5 min (including process if on-demand) | < 2 min (frozen model) |
| Maintenance | Tests update when measures change | Auto-generated test stubs from TMDL |
| Determinism | All tests pass on every run with no flakiness | Same |

## Out of Scope

- Report-level visual testing (no viable automation for PBIRS live-connection reports)
- Performance benchmarking (covered separately in performance-end-to-end.md)
- TE2 BPA configuration (already operational)
- RLS/role-specific testing (future phase — requires `EffectiveUserName` connection string; note as enhancement)

## Risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| ADOMD.NET auth fails from build agent | Medium | R0 validates first; budget 1h for troubleshooting |
| ProcessFull blows the 5-min budget | Medium | Measure during PoC; consider frozen model if > 90s |
| Golden results are a maintenance burden | Medium | Start with < 10 measures; evaluate JSON format reviewability |
| SSAS cache creates false confidence | Low | Disable parallelization; document warm vs cold |

## Expected Output (Phase 1 — 6h PoC)

1. Proof-of-concept xUnit project with:
   - R0 validated: successful ADOMD.NET connection from build agent
   - 2-3 DAX measure tests (including one time-intelligence with pinned context)
   - 1-2 SP execution tests (including one validation beyond row counts)
   - Measured timings: cold connection, warm query, ProcessFull (if applicable)
2. Documented `dotnet test` command for ADO Classic pipeline
3. Recommendation: adopt / adapt / abandon
4. If adopt: proposed test project structure for the repo

## Follow-Up (Phase 2 — if adopted)

- Full test database reset strategy (DACPAC + fixture seed automation)
- SSAS processing gate in pipeline
- Expand coverage to all measures + all load SPs
- RLS testing with `EffectiveUserName`
- TMDL-driven test stub generation
- Golden result versioning and PR review workflow
