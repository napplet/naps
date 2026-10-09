# NAAT-FEED-MANAGER · `feed-manager`

**Recommended open contract:** (none yet)
**Actions:** open

A feed-manager napplet creates, edits, saves, imports, exports, or organizes feed
definitions.

- **IS:** feed definition management: saved filter sets, named feeds, imported
  or exported feed definitions, and tools for organizing the criteria that can
  later drive a feed.
- **IS NOT:** a scrolling timeline or reader for many events by criteria
  (→ `feed`); the payload or lifecycle semantics for opening a specific saved
  feed definition (→ a future convention).
- **Distinct from:** `feed` by authorship vs. consumption: `feed-manager`
  manages feed definitions, while `feed` displays events selected by criteria.

> A NAAT file is this fixed schema and nothing else. Anything that wants to grow —
> payload detail, lifecycle, action-specific wire — belongs in conventions, not here.
> See [../ARCHETYPES.md](../ARCHETYPES.md) for the registry and [../naps/NAP-INTENT.md](../naps/NAP-INTENT.md) for how roles are opened.
