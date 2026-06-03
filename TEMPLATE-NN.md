NAP-{NN}
========

{Title}
-------

`draft`

**NAP ID:** NAP-{NN}
**Domain:** {e.g., feed rendering, chat, collaborative editing}
**Requires:** {NAP-WORD interfaces needed, e.g., NAP-RELAY, NAP-IPC}
**Discovery:** `shell.supports("NAP-{WORD}", "NAP-{NN}")`

## Description

{One paragraph: what event semantics this protocol defines and for what purpose.}

## Event Semantics

{Define the event kinds, tags, and content structure that participating napplets
agree on.}

### Event: {name}

```json
{
  "kind": {kind},
  "tags": [{tag structure}],
  "content": "{content structure}"
}
```

{Behavioral requirements for producers and consumers.}

## Negotiation

{How napplets discover peers that support this protocol.}

## Implementations

- {links to implementations}
