# Requirements: NAP Rename

**Defined:** 2026-06-03
**Core Value:** Every public protocol reference must consistently use NAP: Nostr Applet Protocol, with no remaining legacy acronym or legacy expanded phrase mentions.

## v0.1 Requirements

### Inventory

- [ ] **INV-01**: Maintainer can see a complete inventory of local default-branch files containing `legacy acronym`, `legacy expanded phrase`, `legacy expanded phrase`, the repository directory, or legacy branch prefix.
- [ ] **INV-02**: Maintainer can see a complete inventory of open pull requests whose title, body, branch name, or changed documents contain stale legacy terminology.
- [ ] **INV-03**: Maintainer can verify GitHub issues contain no stale legacy terminology, or see the exact reason issue verification is empty.

### Local Documents

- [ ] **DOC-01**: Reader sees `NAP: Nostr Applet Protocol` as the repository title and concept in `README.md`.
- [ ] **DOC-02**: Reader sees NIP-5D prose, terminology tables, negotiation sections, and extension framework sections use NAP terminology in `SPEC.md`.
- [ ] **DOC-03**: Contributor instructions in `CLAUDE.md` describe NAP proposal tracks, PR preparation, and modification checklists without legacy acronym or legacy expanded phrase.
- [ ] **DOC-04**: Proposal templates use NAP identifiers and examples instead of legacy identifiers and examples.

### Filenames

- [ ] **FILE-01**: Interface proposal template filenames and headings use NAP-oriented names.
- [ ] **FILE-02**: Message protocol proposal template filenames and headings use NAP-oriented names.
- [ ] **FILE-03**: No repository filename in milestone scope contains stale `legacy acronym` or legacy lowercase token naming after the rename.

### Pull Requests

- [ ] **PR-01**: Every open pull request title uses NAP terminology.
- [ ] **PR-02**: Every open pull request body uses NAP terminology and preserves the substantive spec summary, compatibility notes, validation notes, non-goals, and follow-ups.
- [ ] **PR-03**: Every document changed in every open pull request branch uses NAP terminology and filenames where applicable.
- [ ] **PR-04**: Open pull request branch updates are pushed without private implementation references.

### Verification

- [ ] **VER-01**: Final local grep for `legacy acronym`, `legacy expanded phrase`, and `legacy expanded phrase` returns zero matches in tracked repository files, excluding `.git`.
- [ ] **VER-02**: Final filename search returns zero tracked filenames containing `legacy acronym` or legacy lowercase token in milestone scope.
- [ ] **VER-03**: Final GitHub PR metadata check returns zero open pull request titles or bodies containing stale legacy terminology.
- [ ] **VER-04**: Final open pull request branch-content check returns zero stale legacy terminology in documents within those PRs.

## Future Requirements

### Archive Hygiene

- **ARCH-01**: Closed and merged pull request metadata can be rewritten or annotated if the maintainer decides historical review pages must also be terminology-clean.

### Protocol Compatibility

- **COMP-01**: Runtime capability prefixes such as the legacy runtime capability prefix can be renamed only after an explicit compatibility decision and migration plan.

## Out of Scope

| Feature | Reason |
|---------|--------|
| Sibling repository package renames | User scoped this milestone to this repository's documents, filenames, issues, and PR surfaces. |
| Closed/merged PR metadata rewrite | Historical PRs may be archival; keep as future requirement unless explicitly promoted. |
| Runtime wire-prefix migration | the legacy runtime capability prefix may be a compatibility-sensitive protocol value, not just prose branding. |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INV-01 | Phase 1 | Pending |
| INV-02 | Phase 1 | Pending |
| INV-03 | Phase 1 | Pending |
| DOC-01 | Phase 2 | Pending |
| DOC-02 | Phase 2 | Pending |
| DOC-03 | Phase 2 | Pending |
| DOC-04 | Phase 2 | Pending |
| FILE-01 | Phase 2 | Pending |
| FILE-02 | Phase 2 | Pending |
| FILE-03 | Phase 2 | Pending |
| PR-01 | Phase 3 | Pending |
| PR-02 | Phase 3 | Pending |
| PR-03 | Phase 3 | Pending |
| PR-04 | Phase 3 | Pending |
| VER-01 | Phase 4 | Pending |
| VER-02 | Phase 4 | Pending |
| VER-03 | Phase 4 | Pending |
| VER-04 | Phase 4 | Pending |

**Coverage:**
- v0.1 requirements: 18 total
- Mapped to phases: 18
- Unmapped: 0

---
*Requirements defined: 2026-06-03*
*Last updated: 2026-06-03 after roadmap creation*
