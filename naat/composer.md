# NAAT-COMPOSER · `composer`

**Recommended open contract:** (none yet)
**Actions:** open

A napplet that creates a new Nostr event — a reply, quote, new note, or other publishable content.

- **IS:** authoring and publishing a new event, optionally seeded with reply/quote context.
- **IS NOT:** viewing existing events (→ `note` / `feed`); editing app configuration.
- **Distinct from:** `note` (writing vs. reading).

> A NAAT file is this fixed schema and nothing else. Anything that wants to grow —
> payload detail, lifecycle, action-specific wire — belongs in a NAP-N spec, not here.
> See [../ARCHETYPES.md](../ARCHETYPES.md) for the registry and [../NAP-OPEN.md](../NAP-OPEN.md) for how roles are opened.
