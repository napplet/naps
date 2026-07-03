NAP-2
======

Stream Topics
-------------

`draft`

**NAP ID:** NAP-2

**Domain:** stream topic coordination

**Depends:**
- `inc` — capability · required — rides NAP-INC transport (`inc.subscribe` / `inc.event`)

**Discovery:** `shell.supports("inc", "NAP-2")`

## Description

This protocol defines a coherent `stream:*` topic family for napplets that
coordinate active stream selection and stream context through NAP-INC. The
current revision defines:

- `stream:channel-switch`
- `stream:current-context-get`
- `stream:current-context`

This protocol specifies topic strings, payload shapes, producer behavior, and
consumer behavior. It does not redefine NAP-INC transport methods.

## Message Protocol

Napplets coordinate through NAP-INC. The INC event topic is carried by the
event's `t` tag and the payload is carried as JSON in the event content. This
NAP defines the meaning of selected `stream:*` topics; `inc.emit`,
`inc.subscribe`, `inc.unsubscribe`, and `inc.event` remain generic NAP-INC
transport.

Topics in this family MUST use the `stream:` prefix and MUST describe stream
selection, stream context discovery, or companion stream UI synchronization.

### `stream:channel-switch`

Producers send `stream:channel-switch` after the active stream selection changes
and a companion napplet may need to update its context.

Consumers receive `stream:channel-switch` when they can open, focus, or update a
stream-related view such as live chat, metadata, or another companion UI.

Payload:

```json
{
  "streamId": "30311:<hex-pubkey>:<d-tag>",
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
| `streamId` | string | yes | Stable identifier for the selected stream. When the stream is a Nostr addressable event, this SHOULD be its address coordinate. |
| `streamUrl` | string | no | Playback URL hint for the selected stream. |
| `metadata` | object | no | Optional display and routing hints for companion views. |
| `metadata.title` | string | no | Human-readable stream title hint. |
| `metadata.chatRelays` | string[] | no | Relay URL hints for stream chat events. |
| `metadata.image` | string | no | Poster or preview image URL hint. |
| `metadata.hostPubkey` | string | no | Hex Nostr public key hint for the stream host. |

Producer requirements:

- Producers MUST include `streamId`.
- Producers SHOULD include `metadata.chatRelays` when companion views need relay
  hints to discover stream chat.
- Producers MAY include `streamUrl` and display metadata as hints.
- Producers MUST NOT require a specific shell windowing policy, player
  implementation, or companion napplet implementation.

Consumer requirements:

- Consumers MUST ignore payloads without a valid `streamId`.
- Consumers SHOULD update their active stream context to `streamId`.
- Consumers SHOULD treat `metadata` as hints and MAY refresh authoritative stream
  metadata from the stream event or another trusted source.
- Consumers MAY ignore the message when stream companion behavior is unsupported
  in the current context.

### `stream:current-context-get`

Producers send `stream:current-context-get` when they need another napplet to
report its current stream context, for example after a companion view starts and
needs to synchronize state.

Consumers receive `stream:current-context-get` when they can report an active
stream context. Consumers that support this protocol SHOULD answer on
`stream:current-context`.

Payload:

```json
{
  "requestId": "optional-correlation-id"
}
```

Fields:

| Field | Type | Required | Semantics |
|-------|------|----------|-----------|
| `requestId` | string | no | Opaque producer-selected correlation identifier. Responders SHOULD echo it when emitting a response. |

Producer requirements:

- Producers MAY send an empty JSON object.
- Producers MAY include `requestId` when they need to correlate responses.
- Producers MUST treat zero responses as a valid outcome.
- Producers MUST NOT require a specific stream napplet implementation or shell
  focus policy.

Consumer requirements:

- Consumers SHOULD ignore unknown fields.
- Consumers SHOULD answer with the current stream context when one is available.
- Consumers SHOULD echo `requestId` in the response when present.
- Consumers MAY ignore the message when no stream context is available or when
  stream context reporting is unsupported.

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

## Compatibility Alias

Existing implementations may use `livestream:channel-switch` for the same
payload and behavior as `stream:channel-switch`. Consumers MAY accept
`livestream:channel-switch` as a compatibility alias. Producers that advertise
NAP-2 support SHOULD emit `stream:channel-switch`.

The alias is not a separate topic family and does not reserve the broader
`livestream:*` namespace.

## Growth and Competing Drafts

This NAP defines one stream-topic coordination surface. Future numbered NAP-N
drafts MAY define alternate or expanded `stream:*` topic sets. Napplets discover
the specific numbered protocol with `shell.supports("inc", "NAP-2")`; ecosystem
preference between competing stream-topic NAPs is determined by implementation
adoption.

## Negotiation

Napplets discover support for this protocol with:

```
window.napplet.shell.supports("inc", "NAP-2");
```

A napplet that requires INC transport declares the interface dependency in its
manifest:

```
["requires", "inc"]
```

The manifest declares only the interface dependency. The numbered protocol is
negotiated at runtime with `shell.supports()`.

## Non-goals

- Defining generic INC transport.
- Defining media playback control or shell-owned playback routing.
- Defining stream event kind semantics, relay discovery rules, or chat message
  format.
- Defining shell window placement, focus, or lifecycle policy.
- Reserving non-stream topic families such as `profile:*`, `chat:*`, `wm:*`, or
  `shell:*`.

## Implementations

- (none yet)

## Changelog

- `8c9b89c` - Introduced NAP-2 as the stream topic family.
