napplet:{archetype}/{intent} Convention
=======================================

{Title}
-------

`draft`

**Convention:** `napplet:{archetype}/{intent}`
**Invocation:** `napplet:{archetype}/{intent}[...?params]`
**Depends:** {omit when carrier-independent; otherwise list semantic capability dependencies by domain}
- {e.g. `relay` when behavior calls `relay.query`; `inc` and `intent` are carriers, not dependencies}
**Archetype:** {optional — role + action this convention commonly serves, e.g. `note/open`. See ARCHETYPES.md.}

## Description

{One paragraph: what message semantics this convention defines and for what purpose. Describe the coordination pattern between napplets.}

## Payload

Messages follow the carrier used by the caller and handler. When delivered over
NAP-INC or a projection wire, envelopes use `domain.action` form. This convention
defines the semantic meaning of the payload content.

The convention identity is queryless. Invocation query parameters are shallow
payload sugar: the runtime binding percent-decodes unique `name=value` pairs as
text payload fields without scalar coercion. This convention MUST name every
supported query field. Structured or non-text data uses the explicit payload.

### Schemas

{Define records and enums as tables: field, required, type, and notes.}

## Behavior

{Behavioral requirements for producers and consumers.}

## Discovery

Napplets discover the stable, queryless convention identity through handler
metadata, usually:

```
["archetype", "{slug}", "napplet:{archetype}/{intent}"]
```

or through `intent.available()` candidate `conventions`.

## Implementations

- {links to implementations}
