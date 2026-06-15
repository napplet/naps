NAP-{N}
========

{Title}
-------

`draft`

**NAP ID:** NAP-{N}
**Domain:** {e.g., feed rendering, chat, collaborative editing}
**Requires:** {NAP-WORD interfaces needed, e.g., NAP-RELAY, NAP-INC}
**Serves:** {optional — if this is an archetype open contract, the role + verb it serves, e.g. `note/open`. See ARCHETYPES.md. Omit for pure peer-coordination protocols.}
**Discovery:** `shell.supports("{word}", "{n}")`

## Description

{One paragraph: what message semantics this protocol defines and for what purpose. Describe the coordination pattern between napplets.}

## Message Protocol

Napplets coordinate using messages published and received via NAP-RELAY or NAP-INC. Messages follow the NIP-5D wire format (`{ "type": "domain.action", ...payload }`). The protocol defines the semantic meaning of message content -- what napplets agree on when they exchange data.

### {Message Name}

{Description of this message type: when it is sent, what it means, who produces and consumes it.}

When published via NAP-RELAY, the event carries a NIP-01 kind, content, and tags as defined by this protocol. The payload fields and their semantics are defined here.

{Behavioral requirements for producers and consumers.}

## Negotiation

Napplets discover peers supporting this protocol via `shell.supports("{word}", "{n}")`. A napplet requiring this protocol declares it in its manifest:

```
["requires", "{word}"]
```

The protocol number is negotiated at runtime -- the manifest declares the interface dependency, and napplets check for protocol support via `shell.supports()`.

## Implementations

- {links to implementations}
