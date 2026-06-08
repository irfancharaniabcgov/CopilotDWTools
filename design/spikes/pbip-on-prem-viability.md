# Technical Spike: PBIP Format Viability for On-Premises PBIRS

**Status**: Not started
**Time-box**: 4 hours
**Created**: 2026-06-08
**Owner**: TBD

---

## Problem Statement

Power BI reports (.pbix) are stored as binary blobs in Git. This prevents:
- Diffing between versions (no visibility into what changed)
- Merging concurrent edits (single-author workflow required)
- Code review of visual/layout changes in pull requests

The PBIP (Power BI Project) format extracts a .pbix into a folder structure with JSON/TMDL files that are individually diffable and mergeable. However, PBIP was designed for Power BI Service (cloud) deployment. This spike investigates whether PBIP can be used as a **development format** with .pbix as the deployment artifact for PBIRS.

---

## Questions to Answer

1. **Does the PBIRS-optimized PBI Desktop support PBIP save/open?**
   - PBIRS ships its own PBI Desktop version (May 2026 currently). Does it support File → Save As → Power BI Project (.pbip)?
   - If not, can the standard PBI Desktop (monthly release) be used for development while the PBIRS-optimized version is used for final .pbix export?

2. **Is the .pbip → .pbix round-trip lossless?**
   - Save as PBIP → edit → Save As .pbix → publish to PBIRS: does anything break?
   - Specifically: live connection to SSAS, bookmarks, drill-through, conditional formatting, themes

3. **What does the PBIP folder structure look like for a live-connection report?**
   - Since there's no embedded model (live connection), is the PBIP structure simpler?
   - Which files change when: a visual is moved, a page is added, a filter is changed, a bookmark is added?

4. **Can PBIP be used without signing in to Power BI Service?**
   - Some PBIP features may require cloud auth. Can it function fully offline / on-prem?

5. **Does PBI-tools (or a successor) offer anything new?**
   - PBI-tools was previously attempted and didn't work. Has it been updated?
   - Are there alternative tools (e.g., `pbi-inspector`, Fabric Git SDK) that work offline?

---

## Acceptance Criteria

- [ ] Confirmed whether PBIRS Desktop supports PBIP format
- [ ] If yes: documented the round-trip workflow (PBIP for dev → .pbix for deploy)
- [ ] If yes: documented what files change for common report edits (visual add/move, filter change, bookmark)
- [ ] If no: documented the blocker and any workarounds
- [ ] Recommendation: adopt PBIP for dev, stay with binary .pbix, or wait for future PBIRS release

---

## Investigation Steps

1. Open PBIRS-optimized PBI Desktop (May 2026 version)
2. Attempt: File → Save As → look for "Power BI Project (.pbip)" option
3. If available: save an existing live-connection report as PBIP
4. Examine folder structure (especially `report/` folder — pages, visuals, bookmarks)
5. Make a small change (add a visual, change a filter)
6. Observe which files changed (`git diff`)
7. Save back as .pbix
8. Publish to PBIRS and verify: all pages, visuals, drill-through, bookmarks, connection work
9. Document findings

---

## Constraints

- Must work fully on-premises (no Power BI Service dependency)
- Must work with SSAS Tabular live connection
- Must not require additional licensing beyond existing PBIRS + PBI Desktop
- PBI-tools is ruled out (previously attempted, did not work in this environment)

---

## Outcome

_To be filled after investigation._
