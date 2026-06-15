# NAAT-NOTE · `note`

**Recommended open contract:** NAP-4 (`note:open`)
**Actions:** open

A napplet that displays a single Nostr event in focused detail — the event, its author, and its reply thread.

- **IS:** rendering one event and navigating its thread/replies.
- **IS NOT:** a list of many events (→ `feed`); composing or editing (→ `composer`); a profile view (→ `profile`).
- **Distinct from:** `feed` (one event vs. many) · `profile` (an event vs. an author).

> A NAAT file is this fixed schema and nothing else. Anything that wants to grow —
> payload detail, lifecycle, action-specific wire — belongs in a NAP-N spec, not here.
> See [../ARCHETYPES.md](../ARCHETYPES.md) for the registry and [../naps/NAP-INTENT.md](../naps/NAP-INTENT.md) for how roles are opened.
