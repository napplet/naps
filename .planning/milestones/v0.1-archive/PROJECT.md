# NAP: Nostr Applet Protocol

## What This Is

NAP is the extension proposal system for the Nostr applet protocol. This repository hosts the governance document, templates, and proposal specs that define applet-facing shell interfaces and inter-applet message protocols around NIP-5D.

## Core Value

Every public protocol reference must consistently use NAP: Nostr Applet Protocol, with no remaining legacy acronym or legacy expanded phrase mentions.

## Current Milestone: v0.1 Rename legacy acronym to NAP

**Goal:** Rename the public protocol vocabulary from the legacy acronym to NAP across repository documents, filenames, GitHub pull request metadata, and branch documents.

**Target features:**
- Replace all local default-branch document text that says legacy acronym or legacy expanded phrase with NAP or Nostr Applet Protocol.
- Rename every local legacy-derived filename and template placeholder to the NAP naming scheme.
- Update all open GitHub pull request titles, bodies, and branch documents so the public review surface uses NAP.
- Verify issues contain no stale terminology; issues are disabled on this repository, so verification is expected to be empty.
- End with zero matches for legacy acronym and legacy expanded phrase across local docs and reachable GitHub PR surfaces.

## Requirements

### Validated

- Repository currently contains the NIP-5D applet extension governance document, proposal templates, and draft protocol/spec text.
- GitHub pull requests are the review surface for individual interface and message protocol specs.
- GitHub issues are disabled for `napplet/naps`.

### Active

- [ ] Rename public protocol terminology from the legacy acronym to NAP across local documents.
- [ ] Rename public filenames and template identifiers from legacy-based names to NAP-based names.
- [ ] Update open GitHub pull request titles and bodies to NAP terminology.
- [ ] Update every document inside every open pull request branch so branch diffs do not reintroduce legacy terminology.
- [ ] Verify GitHub issue surfaces contain no stale terminology.
- [ ] Produce a final audit showing no legacy acronym or legacy expanded phrase references remain in milestone scope.

### Out of Scope

- Renaming the the legacy runtime capability prefix runtime capability prefix without explicit protocol decision -- this may be wire compatibility rather than prose branding.
- Updating closed or merged pull request metadata unless a later execution phase explicitly decides archival metadata must be rewritten -- GitHub history may intentionally preserve old review text.
- Changing implementation packages in sibling repositories -- this milestone targets this specs repository and its GitHub review surface.

## Context

- Local working tree is `this repository checkout`.
- Git remote resolves through GitHub CLI as `napplet/naps`, default branch `master`.
- The repository currently has no committed `.planning` directory; this milestone creates the planning baseline.
- Local default branch stale terminology appears in `README.md`, `SPEC.md`, `CLAUDE.md`, `TEMPLATE-WORD.md`, and `TEMPLATE-NN.md`.
- GitHub has 18 open pull requests with legacy-derived titles and branch names as of 2026-06-03.
- `gh issue list` reports that issues are disabled for the repository.

## Constraints

- **Public repository hygiene:** Specs, PR bodies, and commits must not mention private implementation repositories or packages.
- **Review surface:** Open PR titles and bodies must be updated with `gh pr edit`; branch documents require branch checkout or worktree handling.
- **Compatibility:** Runtime strings such as the legacy runtime capability prefix require explicit judgment before mechanical replacement because they may be protocol wire values.
- **Verification:** Final acceptance requires grep-style checks for `legacy acronym`, `legacy expanded phrase`, and `legacy expanded phrase` across local docs and reachable PR surfaces.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Use NAP as the replacement acronym | User specified `NAP: Nostr Applet Protocol` | Pending |
| Treat issues as verification-only | GitHub reports issues disabled for this repository | Pending |
| Skip domain research | This is a terminology migration, not a new feature domain | Pending |
| Keep closed/merged PR metadata out of initial scope | User asked every PR surface, but open PRs are actionable review documents while closed history may be archival | Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `$gsd-transition`):
1. Requirements invalidated? -> Move to Out of Scope with reason
2. Requirements validated? -> Move to Validated with phase reference
3. New requirements emerged? -> Add to Active
4. Decisions to log? -> Add to Key Decisions
5. "What This Is" still accurate? -> Update if drifted

**After each milestone** (via `$gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check -- still the right priority?
3. Audit Out of Scope -- reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-06-03 after new milestone initialization*
