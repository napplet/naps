# Phase 1: Inventory and Safety Boundaries - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase inventories stale public protocol vocabulary in the default branch, active GitHub pull requests, and issue surfaces before rename edits begin.

</domain>

<decisions>
## Implementation Decisions

### the agent's Discretion
- Treat this as an infrastructure phase with no user-facing UX choices.
- Keep raw stale-token command output outside planning artifacts so milestone files do not preserve forbidden strings.
- Record actionable counts, file names, and remediation policy in planning artifacts.
- Classify the legacy runtime capability prefix as in-scope for migration because the user requested no legacy vocabulary remain.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- GitHub CLI is authenticated and can list PRs, PR files, and issue state.
- Ripgrep is available for local inventory.

### Established Patterns
- The repository is documentation/specification-only at the default branch surface.
- `.planning/` is ignored by `.gitignore`; GSD artifacts must be force-added when committed.

### Integration Points
- Local docs: `README.md`, `SPEC.md`, `CLAUDE.md`, `TEMPLATE-WORD.md`, `TEMPLATE-NN.md`.
- Active PRs: 18 open pull requests with stale title vocabulary and changed spec documents.

</code_context>

<specifics>
## Specific Ideas

The user requested a complete rename to `NAP: Nostr Applet Protocol` across documents, filenames, issues, PR titles, PR bodies, and strings inside documents within every PR.

</specifics>

<deferred>
## Deferred Ideas

Closed and merged PR metadata remains deferred unless explicitly promoted.

</deferred>
