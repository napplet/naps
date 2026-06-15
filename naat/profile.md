# NAAT-PROFILE · `profile`

**Recommended open contract:** NAP-1 (`profile:open`)
**Actions:** open

A napplet that displays a user's profile — metadata, and a view of their activity or relationships.

- **IS:** rendering one pubkey's identity and activity.
- **IS NOT:** a single note (→ `note`); a direct-message conversation (→ `dm`).
- **Distinct from:** `note` (an author vs. an event) · `dm` (viewing a person vs. conversing with them).

> A NAAT file is this fixed schema and nothing else. Anything that wants to grow —
> payload detail, lifecycle, action-specific wire — belongs in a NAP-N spec, not here.
> See [../ARCHETYPES.md](../ARCHETYPES.md) for the registry and [../naps/NAP-INTENT.md](../naps/NAP-INTENT.md) for how roles are opened.
