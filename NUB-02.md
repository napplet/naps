NUB-02
======

Chat Direct Message Open
------------------------

`draft`

**NUB ID:** NUB-02

**Domain:** chat navigation

**Requires:** NUB-IFC

**Discovery:** `shell.supports("ifc", "NUB-02")`

## Description

This protocol defines the `chat:open-dm` topic semantics for napplets that want
to request that another napplet open or focus a direct-message conversation with
a Nostr public key. It specifies the topic string, payload shape, producer
behavior, and consumer behavior. It does not redefine NUB-IFC transport methods.

## Message Protocol

Napplets coordinate through NUB-IFC. The IFC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NUB defines the meaning of one IFC topic; `ifc.emit`, `ifc.subscribe`,
`ifc.unsubscribe`, and `ifc.event` remain generic NUB-IFC transport.

### `chat:open-dm`

Producers send `chat:open-dm` when a user action selects a person or identity
for direct conversation, for example from a profile view, contact list, mention,
or message action.

Consumers receive `chat:open-dm` when they can open, focus, or otherwise present
a direct-message conversation for the requested public key.

Payload:

```json
{
  "pubkey": "<hex-pubkey>",
  "displayName": "Ada"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `pubkey` | string | yes | 64-character lowercase hexadecimal Nostr public key for the conversation peer. |
| `displayName` | string | no | Human-readable label hint for optimistic UI. Consumers MUST NOT treat it as authoritative identity metadata. |

Producer requirements:

- Producers MUST include `pubkey`.
- Producers SHOULD use canonical lowercase hexadecimal public keys.
- Producers MAY include `displayName` as a display hint.
- Producers MUST NOT require a specific shell windowing policy or chat napplet
  implementation.

Consumer requirements:

- Consumers MUST ignore payloads without a valid `pubkey`.
- Consumers SHOULD open or focus a direct-message conversation for `pubkey`.
- Consumers MAY show `displayName` while resolving profile metadata, but MUST
  treat profile events or local identity state as authoritative.
- Consumers MAY ignore the message when direct-message navigation is unsupported
  in the current context.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("ifc", "NUB-02");
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
- Defining message encryption, gift wrap, signing, relay selection, or chat
  history retrieval.
- Defining shell window placement, focus, or routing policy.

## Implementations

- (none yet)
