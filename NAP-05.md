NAP-05
======

Stream Current Context
----------------------

`draft`

**NAP ID:** NAP-05

**Domain:** stream context reporting

**Requires:** NAP-IFC

**Discovery:** `shell.supports("ifc", "NAP-05")`

## Description

This protocol defines the `stream:current-context` topic semantics for napplets
that report an active stream context to other napplets. It specifies the topic
string, payload shape, producer behavior, and consumer behavior. It does not
redefine NAP-IFC transport methods.

## Message Protocol

Napplets coordinate through NAP-IFC. The IFC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of one IFC topic; `ifc.emit`, `ifc.subscribe`,
`ifc.unsubscribe`, and `ifc.event` remain generic NAP-IFC transport.

### `stream:current-context`

Producers send `stream:current-context` when reporting their current stream
context, either in response to a request or as an unsolicited state update.

Consumers receive `stream:current-context` when they can synchronize a companion
view, such as chat, metadata, or other stream-aware UI, to the reported context.

Payload:

```json
{
  "streamAddr": "30311:<hex-pubkey>:<d-tag>",
  "title": "Live stream title",
  "chatRelays": ["wss://relay.example.test"],
  "requestId": "optional-correlation-id"
}
```

Empty-state payload:

```json
{
  "streamAddr": null,
  "title": null,
  "chatRelays": [],
  "requestId": "optional-correlation-id"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `streamAddr` | string or null | yes | Active stream address coordinate, or `null` when no active stream context exists. |
| `title` | string or null | no | Human-readable title hint for the active stream, or `null` when unavailable. |
| `chatRelays` | string[] | yes | Relay URL hints for stream chat. Empty when unknown or when no active stream exists. |
| `requestId` | string | no | Opaque correlation identifier copied from a request topic when present. |

Producer requirements:

- Producers MUST include `streamAddr`.
- Producers MUST include `chatRelays`, using an empty array when no relay hints
  are available.
- Producers SHOULD set `streamAddr` to `null`, `title` to `null`, and
  `chatRelays` to `[]` when reporting that no active stream exists.
- Producers SHOULD echo `requestId` when responding to a request that supplied
  one.
- Producers MUST NOT require a specific shell focus policy or companion napplet
  implementation.

Consumer requirements:

- Consumers MUST ignore payloads that omit `streamAddr` or `chatRelays`.
- Consumers SHOULD treat `streamAddr: null` as an explicit no-active-stream
  state.
- Consumers SHOULD use `chatRelays` as relay hints and MAY refresh authoritative
  stream metadata from the stream event or another trusted source.
- Consumers MAY ignore responses with a `requestId` that does not match their
  pending request.
- Consumers MAY accept unsolicited updates when useful for their UI.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("ifc", "NAP-05");
```

A napplet that requires IFC transport declares the interface dependency in its
manifest:

```
["requires", "ifc"]
```

The manifest declares only the interface dependency. The numbered protocol is
negotiated at runtime with `shell.supports()`.

## Non-goals

- Defining generic IFC transport.
- Defining the request payload shape for `stream:current-context-get`.
- Defining livestream channel switch semantics.
- Defining stream event kind semantics, relay discovery rules, or chat message
  format.

## Implementations

- (none yet)
