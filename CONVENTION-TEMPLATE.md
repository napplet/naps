{topic}:{action} Convention
===========================

{Title}
-------

`draft`

**Convention:** `{topic}:{action}`
**Depends:** {by domain when a runtime capability is required}
- `inc` — capability · optional — may be delivered as an INC topic event.
- {any further domains, e.g. `relay` when the payload expects relay-backed lookup}
**Archetype:** {optional — role + action this convention commonly serves, e.g. `note/open`. See ARCHETYPES.md.}

## Description

{One paragraph: what message semantics this convention defines and for what purpose. Describe the coordination pattern between napplets.}

## Payload

Messages follow the carrier used by the caller and handler. When delivered over
NAP-INC or a projection wire, envelopes use `domain.action` form. This convention
defines the semantic meaning of the payload content.

### Schemas

{Define records and enums in CDDL-style notation.}

## Behavior

{Behavioral requirements for producers and consumers.}

## Discovery

Napplets discover support for this convention through handler metadata, usually:

```
["archetype", "{slug}", "{topic}:{action}"]
```

or through `intent.available()` candidate `conventions`.

## Implementations

- {links to implementations}
