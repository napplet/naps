# Phase 2: Local Default-Branch Rename - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase updates the default branch documents and filenames so the repository itself presents NAP consistently.

</domain>

<decisions>
## Implementation Decisions

### Local Rename Policy
- Replace the legacy acronym and expanded phrase with `NAP` and `Nostr Applet Protocol`.
- Replace the legacy runtime capability prefix with the NAP runtime prefix in documentation examples.
- Update repository links from the old repository path to `napplet/naps`.
- Rename proposal templates to NAP-oriented filenames and update all local references.

### the agent's Discretion
- Keep edits mechanical and scoped to terminology, filenames, and links.
- Avoid broad prose rewrites unless needed to keep the renamed text coherent.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- Local branch contains only documentation/spec files.

### Established Patterns
- README holds governance and registry rows.
- SPEC holds NIP-5D protocol framing and capability query examples.
- CLAUDE holds contributor instructions.
- Templates define proposal skeletons.

### Integration Points
- `README.md`, `SPEC.md`, `CLAUDE.md`, `NAP-WORD-TEMPLATE.md`, and `NAP-N-TEMPLATE.md`.

</code_context>

<specifics>
## Specific Ideas

Use `NAP: Nostr Applet Protocol` as the canonical title and expansion.

</specifics>

<deferred>
## Deferred Ideas

None.

</deferred>
