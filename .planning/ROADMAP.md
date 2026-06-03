# Roadmap: NAP Rename

## Overview

This milestone converts the public specs repository and its active GitHub review surface from legacy terminology to NAP: Nostr Applet Protocol. The work starts with an inventory, then updates default-branch docs and filenames, then cleans open pull request metadata and branch documents, and ends with zero-match verification across the agreed scope.

## Phases

**Phase Numbering:**
- Integer phases are planned milestone work.
- This repository had no prior `.planning` state, so numbering starts at Phase 1.

- [x] **Phase 1: Inventory and Safety Boundaries** - Build the exact local, issue, and open-PR stale-term inventory before editing.
- [ ] **Phase 2: Local Default-Branch Rename** - Rename repository documents, templates, and filenames on `master`.
- [ ] **Phase 3: Open Pull Request Surface Cleanup** - Update open PR titles, bodies, and changed branch documents.
- [ ] **Phase 4: Final Audit and Handoff** - Prove zero stale terminology remains in milestone scope and document any explicit exceptions.

## Phase Details

### Phase 1: Inventory and Safety Boundaries
**Goal**: Produce a complete inventory and identify compatibility-sensitive strings before any rename edits.
**Depends on**: Nothing
**Requirements**: [INV-01, INV-02, INV-03]
**Success Criteria** (what must be TRUE):
  1. Local file inventory lists every tracked document containing stale terminology.
  2. Open PR inventory lists every affected PR title, body, branch name, and changed document.
  3. Issue check records that issues are disabled, or lists matching issue surfaces if GitHub state changes.
  4. Compatibility-sensitive strings such as the legacy runtime capability prefix are explicitly classified before replacement.
**Plans**: 1 plan

Plans:
- [ ] 01-01: Build rename inventory and replacement policy

### Phase 2: Local Default-Branch Rename
**Goal**: Update the default branch documents and filenames so the repository itself presents NAP consistently.
**Depends on**: Phase 1
**Requirements**: [DOC-01, DOC-02, DOC-03, DOC-04, FILE-01, FILE-02, FILE-03]
**Success Criteria** (what must be TRUE):
  1. `README.md`, `SPEC.md`, and `CLAUDE.md` use NAP terminology and no longer mention legacy acronym or legacy expanded phrase.
  2. Proposal templates use NAP identifiers, examples, and headings.
  3. Template filenames and any other tracked legacy-derived filenames are renamed to NAP-derived names.
  4. Local tracked-file grep and filename search pass before moving to PR cleanup.
**Plans**: 2 plans

Plans:
- [ ] 02-01: Rename local document prose and template content
- [ ] 02-02: Rename local files and update references

### Phase 3: Open Pull Request Surface Cleanup
**Goal**: Clean the active GitHub review surface so open PR titles, bodies, and branch documents no longer carry stale terminology.
**Depends on**: Phase 2
**Requirements**: [PR-01, PR-02, PR-03, PR-04]
**Success Criteria** (what must be TRUE):
  1. All 18 currently open pull request titles use NAP terminology.
  2. All 18 currently open pull request bodies use NAP terminology while preserving review content.
  3. Each open pull request branch has its changed documents updated to NAP terminology and filenames where applicable.
  4. Branch pushes and PR edits introduce no private implementation references.
**Plans**: 3 plans

Plans:
- [ ] 03-01: Update open PR titles and bodies with `gh pr edit`
- [ ] 03-02: Update open PR branch documents and filenames
- [ ] 03-03: Re-check PR diffs and metadata for stale terminology

### Phase 4: Final Audit and Handoff
**Goal**: Verify the milestone's zero-stale-terminology contract and document exact status.
**Depends on**: Phase 3
**Requirements**: [VER-01, VER-02, VER-03, VER-04]
**Success Criteria** (what must be TRUE):
  1. Local tracked-file grep returns zero stale legacy terminology matches in milestone scope.
  2. Local tracked filename search returns zero stale legacy-derived names in milestone scope.
  3. GitHub open PR title/body search returns zero stale terminology matches.
  4. Open PR branch-content checks return zero stale terminology matches in documents.
  5. Any remaining compatibility-preserved string is documented as an explicit exception with rationale.
**Plans**: 1 plan

Plans:
- [ ] 04-01: Run final audit, record results, and prepare handoff

## Progress

**Execution Order:**
Phases execute in numeric order: 1 -> 2 -> 3 -> 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Inventory and Safety Boundaries | 1/1 | Complete | 2026-06-03 |
| 2. Local Default-Branch Rename | 0/2 | Not started | - |
| 3. Open Pull Request Surface Cleanup | 0/3 | Not started | - |
| 4. Final Audit and Handoff | 0/1 | Not started | - |
