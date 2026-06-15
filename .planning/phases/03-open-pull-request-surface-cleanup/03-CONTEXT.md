# Phase 3: Open Pull Request Surface Cleanup - Context

**Gathered:** 2026-06-03
**Status:** Ready for planning

<domain>
## Phase Boundary

This phase updates active GitHub pull request titles, bodies, and branch documents so the review surface uses NAP terminology.

</domain>

<decisions>
## Implementation Decisions

### PR Update Policy
- Use GitHub CLI to edit PR titles and bodies.
- Use isolated temporary worktrees for branch document updates.
- Commit branch document changes directly to each open PR head branch.
- Verify PR titles, bodies, tracked branch content, and tracked branch filenames after the update.

### the agent's Discretion
- Preserve substantive PR body sections while applying mechanical terminology updates.
- Do not replace PR head branch names; GitHub does not support changing the head branch of an existing PR without replacing the PR.

</decisions>

<code_context>
## Existing Code Insights

### Reusable Assets
- `gh pr list`, `gh pr view`, and `gh pr edit` can manage PR metadata.
- Git worktrees allow safe branch edits without disturbing the milestone branch.

### Established Patterns
- Each spec PR changes one proposal file, sometimes with a README row.
- Branch documents are Markdown specs and can accept the same terminology replacement policy as default-branch docs.

### Integration Points
- 18 open PRs.
- Each open PR head branch on `origin`.

</code_context>

<specifics>
## Specific Ideas

The user specifically requested every PR body, title, and string in every document within every PR be updated.

</specifics>

<deferred>
## Deferred Ideas

PR head branch renames are not included because they are not PR title/body/document content and cannot be changed in place through PR metadata editing.

</deferred>
