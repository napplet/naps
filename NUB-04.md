NUB-04
======

Stream Current Context Request
------------------------------

`draft`

**NUB ID:** NUB-04

**Domain:** stream context discovery

**Requires:** NUB-IFC

**Discovery:** `shell.supports("ifc", "NUB-04")`

## Description

This protocol defines the `stream:current-context-get` topic semantics for
napplets that want to request the currently active stream context from another
napplet. It specifies the topic string, payload shape, producer behavior, and
consumer behavior. It does not redefine NUB-IFC transport methods.

## Message Protocol

Napplets coordinate through NUB-IFC. The IFC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NUB defines the meaning of one IFC topic; `ifc.emit`, `ifc.subscribe`,
`ifc.unsubscribe`, and `ifc.event` remain generic NUB-IFC transport.

### `stream:current-context-get`

Producers send `stream:current-context-get` when they need another napplet to
report its current stream context, for example after focus changes or when a
companion view starts and needs to synchronize state.

Consumers receive `stream:current-context-get` when they can report an active
stream context. Consumers that also support the stream current-context response
protocol SHOULD answer on `stream:current-context`.

Payload:

```json
{
  "requestId": "optional-correlation-id"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `requestId` | string | no | Opaque producer-selected correlation identifier. Responders SHOULD echo it when emitting a response topic that supports request correlation. |

Producer requirements:

- Producers MAY send an empty JSON object.
- Producers MAY include `requestId` when they need to correlate responses.
- Producers MUST treat zero responses as a valid outcome.
- Producers MUST NOT require a specific stream napplet implementation or shell
  focus policy.

Consumer requirements:

- Consumers SHOULD ignore unknown fields.
- Consumers SHOULD answer with the current stream context when they support a
  response topic for that context.
- Consumers SHOULD echo `requestId` in the response when present.
- Consumers MAY ignore the message when no stream context is available or when
  stream context reporting is unsupported.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("ifc", "NUB-04");
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
- Defining the response payload shape for `stream:current-context`.
- Defining shell focus, window, or lifecycle policy.
- Defining livestream channel switch semantics.

## Implementations

- (none yet)
