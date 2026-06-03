NAP-03
======

Livestream Channel Switch
-------------------------

`draft`

**NAP ID:** NAP-03

**Domain:** livestream context coordination

**Requires:** NAP-IFC

**Discovery:** `shell.supports("ifc", "NAP-03")`

## Description

This protocol defines the `livestream:channel-switch` topic semantics for
napplets that want to coordinate an active livestream selection with another
napplet, such as a chat or companion view. It specifies the topic string,
payload shape, producer behavior, and consumer behavior. It does not redefine
NAP-IFC transport methods.

## Message Protocol

Napplets coordinate through NAP-IFC. The IFC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of one IFC topic; `ifc.emit`, `ifc.subscribe`,
`ifc.unsubscribe`, and `ifc.event` remain generic NAP-IFC transport.

### `livestream:channel-switch`

Producers send `livestream:channel-switch` after the active livestream channel
changes and a companion napplet may need to update its context.

Consumers receive `livestream:channel-switch` when they can open, focus, or
update a livestream-related view such as live chat.

Payload:

```json
{
  "streamId": "<stream-address>",
  "streamUrl": "https://example.test/live.m3u8",
  "metadata": {
    "title": "Live stream title",
    "chatRelays": ["wss://relay.example.test"],
    "image": "https://example.test/poster.jpg",
    "hostPubkey": "<hex-pubkey>"
  }
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `streamId` | string | yes | Stable identifier for the selected livestream. When the stream is a Nostr addressable event, this SHOULD be its address coordinate. |
| `streamUrl` | string | no | Playback URL hint for the selected stream. |
| `metadata` | object | no | Optional display and routing hints for companion views. |
| `metadata.title` | string | no | Human-readable stream title hint. |
| `metadata.chatRelays` | string[] | no | Relay URL hints for livestream chat events. |
| `metadata.image` | string | no | Poster or preview image URL hint. |
| `metadata.hostPubkey` | string | no | Hex Nostr public key hint for the stream host. |

Producer requirements:

- Producers MUST include `streamId`.
- Producers SHOULD include `metadata.chatRelays` when companion views need
  relay hints to discover livestream chat.
- Producers MAY include `streamUrl` and display metadata as hints.
- Producers MUST NOT require a specific shell windowing policy, player
  implementation, or chat napplet implementation.

Consumer requirements:

- Consumers MUST ignore payloads without a valid `streamId`.
- Consumers SHOULD update their active livestream context to `streamId`.
- Consumers SHOULD treat `metadata` as hints and MAY refresh authoritative
  stream metadata from the stream event or another trusted source.
- Consumers MAY ignore the message when livestream companion behavior is
  unsupported in the current context.

## Negotiation

Napplets discover support for this protocol with:

```ts
window.napplet.shell.supports("ifc", "NAP-03");
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
- Defining media playback control or shell-owned playback routing.
- Defining livestream event kinds, relay discovery rules, or chat message
  format.
- Promoting `stream:channel-switch` as an alias.

## Implementations

- (none yet)
